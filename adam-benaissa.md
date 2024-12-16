## Installer et sécuriser Nginx et le serveur Debian

J'ai commencé par installer Nginx:
```
root@debian:~# sudo apt install nginx -y


root@debian:~# sudo systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Mon 2024-12-16 14:12:58 CET; 3min 3s ago

```

Je configure les màj automatiques 
```
root@debian:~# dpkg-reconfigure --priority=low unattended-upgrades
```

j'empêche root de se connecter hors ssh pour rendre les attaques brute-force plus difficile.
```
root@debian:~# usermod -s /usr/sbin/nologin root
```

J'installe un Pare-feu, j'ouvre le port 22 pour ssh et j'autorise les connexions http et https pour Nginx. Je vérifie que le part-feu est lancé, il ne l'est pas donc je le lance.
```
root@debian:~# apt install ufw -y

root@debian:~# ufw allow 22
Rules updated
Rules updated (v6)

root@debian:~# ufw allow 80
Rules updated
Rules updated (v6)

root@debian:~# ufw allow 443
Rules updated
Rules updated (v6)

root@debian:~# ufw status verbose
Status: inactive

root@debian:~# ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

J'installe un IPS (fail2ban) pour proteger des attaques bruteforce, je configure les rèlges pour ssh et Ngnix, puis je redémarre Fail2ban
```
root@debian:~# apt install fail2ban -y

root@debian:~# nano /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
action = %(action_mwl)s

[sshd]
enabled = true

[nginx-http-auth]
enabled = true


root@debian:~# systemctl restart fail2ban

```

je créer l'utilisateur pwn404, je lui donne les perms sudo et je desactives les connexions pour le root. Je vérifie que la connexion root est bien desactivée
```
root@debian:~# adduser pwn404
root@debian:~# usermod -aG sudo pwn404

root@debian:~# su - pwn404

pwn404@debian:~$ sudo whoami
[sudo] password for pwn404: 
root

pwn404@debian:~$ sudo nano /etc/ssh/sshd_config
PermitRootLogin no 

pwn404@debian:~$ sudo systemctl restart sshd

pwn404@debian:~$ ssh root@wilfart.fr -p 6252
The authenticity of host '[wilfart.fr]:6252 ([82.65.221.48]:6252)' can't be established.
ED25519 key fingerprint is SHA256:6HDw/64eaih1pepbGeWdsUF1i8RP3mH1Wc0ox0xs/dM.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? y
Please type 'yes', 'no' or the fingerprint: yes
Warning: Permanently added '[wilfart.fr]:6252' (ED25519) to the list of known hosts.
root@wilfart.fr's password: 
Permission denied, please try again.
```

J'installe un antivirus (ClamAV) et je le configure pour qu'il scanne le dossier web de Nginx
```
pwn404@debian:~$ sudo apt install clamav -y
pwn404@debian:~$ sudo clamscan -r /var/www

```

J'installe auditd pour surveiller les modifications du système.
```
pwn404@debian:~$ sudo apt install auditd -y
pwn404@debian:~$ sudo systemctl enable auditd
pwn404@debian:~$ sudo systemctl start auditd
```

J'ai ajouté des en-têtes http dans mon fichier de conf Nginx pour renforcer la sécurité du serveur web
```
pwn404@debian:~$ sudo nano /etc/nginx/nginx.conf
add_header X-Content-Type-Options nosniff;
add_header X-Frame-Options "SAMEORIGIN";
add_header X-XSS-Protection "1; mode=block";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; object-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'; frame-ancestors 'self';";
add_header Referrer-Policy "no-referrer-when-downgrade";

pwn404@debian:~$ sudo systemctl restart nginx
```
