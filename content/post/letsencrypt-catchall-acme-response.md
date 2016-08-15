---
categories: ["nginx", "ssl", "letsencrypt", "ansible", "automation"]
date: 2016-08-11T23:00:00+02:00
description: ""
keywords: []
draft: false
title: Let's Encrypt catchall acme responses
---

... oder: Spaß mit LXC, nginx und ansible

## Vorgeplänkere

Bei [Biocrafting.net](https://www.biocrafting.net) haben wir bisher auf ein Wildcard Zertifikat von Comodo gesetzt.
Allerdings ist dieses vor ein paar Monaten ausgelaufen und angesichts der [Praktiken](https://letsencrypt.org/2016/06/23/defending-our-brand.html) die der Verein an den Tag legt, sehe ich es unter anderem nicht ein, wieder knapp 100€ für das Zertifikat auszugeben.
Da das Zertifikat für 1 Jahr gültig war, habe ich mir nie Gedanken über eine Automatisierung des Vorganges gemacht.
Mit Let's Encrypt sieht es allerdings anders aus, das Zertifikat hat nur eine Gültigkeitsdauer von 3 Monaten. Dazu muss für jede Subdomain eine gültige ACME-Challenge präsentiert werden - eine Herausforderung, wenn die knapp 25 Subdomains auf 3 Server verteilt sind. "Erschwerend" kommt hinzu, dass ich auf der Domain HPKP aktiviert habe - Ich kann also nicht mit jedem beliebigen Private Key ein Zertifikat generieren. Eine zentralisierte Lösung musste her, wo ich den Private Key vorgeben kann.


Daher habe ich mich dazu entschieden einen gesonderten Container zu erzeugen, welcher ausschließlich für die Zertifikatsanforderungen zuständig ist.
So habe ich die komplette Kontrolle über das Deployment und habe auch eine einheitliche SSL-Konfiguration, die vorher auf jedem System unterschiedlich war.

Eckdaten:

* Debian Jessie Container
* Only IPv6 (v4 Adressen sind knapp!)
* Software: [nginx](https://nginx.org), [acme.sh](https://github.com/Neilpang/acme.sh), [Ansible](https://www.ansible.com/)
* Genutzte [Ansible-Rollen](https://github.com/Biocrafting/ansible-roles)
 * letsencrypt-host-prepare-nginx 
 * letsencrypt-host-renew-nginx

Ich habe den Container __le-auth__ getauft. Er bekommt auch eine eigene Subdomain, nämlich `le-auth.biocrafting.net`.
Der Container stellt einen Let's Encrypt Client und einen Webserver (Für das Ausliefern der ACME Auth Challenges) bereit.

Natürlich könnt ihr auch direkt mit der IP arbeiten, oder den Server z. B. über Digital Ocean erstellen.

## Vorbereitungen auf den Webservern

Für jede Domain muss ein Reverse-Proxy eingerichtet werden, damit das `.well-known/acme-challenges/` Verzeichnis zu dem `le-auth.biocrafting.net` Server weitergeleitet wird.

Im Konfigurationsordner von nginx habe ich einfach ein `global` Ordner angelegt, wo ich folgende Datei hinterlegt habe:

letsencrypt-auth.conf

```
location /.well-known/acme-challenge {
    location ~ /.well-known/acme-challenge/(.*) {
        resolver 8.8.8.8;
        proxy_pass http://le-auth.biocrafting.net$request_uri;
        proxy_set_header  Host $host;
        proxy_set_header  X-Real-IP $remote_addr;
    }
}
```

Für jeden VHost muss logischerweise die Konfiguration eingebunden werden. Beispiel:

```
server {
    listen  [::]:443 ssl;
    keepalive_timeout   70;
    server_name ipv6.biocrafting.net;
    
    ...

    include /etc/nginx/global/letsencrypt-auth.conf;
}
```

Danach muss natürlich nginx angewiesen werden die Konfiguration neu einzulesen.


## Vorbereitungen auf dem Containern

#### Benötige Software installieren/DNS Server setzen/Nutzer hinzufügen

```
apt-get install nginx ansible
#Da github leider noch nicht über ipv6 erreichbar ist, benutze ich den NAT64-Service von http://aa.net.uk/
echo "nameserver 2001:8b0:6464::1" > /etc/resolv.conf

adduser --disabled-password acme
mkdir /var/www/acme-challenges
chmod -R 0775 /var/www/acme-challenges
chgrp -R acme /var/www/acme-challenges
su acme --shell=/bin/bash
cd ~acme
git clone https://github.com/Neilpang/acme.sh.git
cd acme.sh
./acme.sh --install
cd ~; git clone https://github.com/Biocrafting/ansible-roles
ssh-keygen -f id_rsa.rsa -t rsa -b 4096 -N ''
# Der SSH-Key wird später benötigt!
cat ~/.ssh/id_rsa.pub
```

#### Ansible konfigurieren

Ansible muss noch wissen, wo die Rollen zu finden sind. Außerdem muss eine "Webservers" Gruppe mit den Webservern erstellt werden.

```
echo "roles_path    = /home/acme/ansible/" >> /etc/ansible/ansible.cfg
```

vim /etc/ansible/hosts

```
[webservers]
wolfram.biocrafting.net
bctest.biocrafting.net
vanadium.biocrafting.net
xenon.biocrafting.net
```


#### nginx vhost anlegen

```
echo "server {
    listen 80;
    listen [::]:80;
    server_name le-auth.biocrafting.net _;
    #Falls gewünscht hier die Netze der Webserver anpassen (Nützlich zur Fehlersuche)
    set_real_ip_from 2001:1608:10:162::/64;
    set_real_ip_from 2001:1608:10:193::/64;
    real_ip_header X-Real-IP;
    real_ip_recursive on;
    default_type text/plain;
    location /.well-known/acme-challenge {
        alias /var/www/acme-challenges/.well-known/acme-challenge;
        location ~ /.well-known/acme-challenge/(.*) {
        default_type text/plain;
        }
    }
}" > /etc/nginx/sites-available/le-auth.biocrafting.net
ln -s /etc/nginx/sites-available/le-auth.biocrafting.net /etc/nginx/sites-enabled/le-auth.biocrafting.net
systemctl reload nginx
```

#### Ansible Rollen anpassen

##### letsencrypt-host-prepare-nginx
Wichtig hierbei sind die Variablen in der Rolle. Standardwerte sind in der `defaults/main.yml` definiert. Sie können dort angepasst werden, oder dem Playbook übergeben werden:

```
acme@le-auth:~/ansible-roles$ cat host-prepare.yml 
---
- hosts: webservers
  roles:
    - letsencrypt-host-prepare-nginx
  vars:
    le:
      user: bcssl
      host: "2001:1608:10:162::100/128"
      key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDK9Ic5QmziTUvPckNrz9KXHcRdfe9bpP7h56I0sOPJcrOlGoNTfDzsKQWUS4EzbLSFQSsIwQBeAgbZTSTbcsVTAKo/AhJyqAPzTLHt6WlWFhHs/hVj+YWKPRUwp4reBMEKxmmagGl5YC7K4lY8oiBFqA2hcLmJYdBFmFphYKSIcT0CS1ucqHGf0xN5odD+CZNpFqkudGmB+vbHGAWTaw2KCgnBTPmtnascTaYJlxVmq0Q2FBh2F++OTLjU2y20NSEpBw/THFd+7znFRxr1J3QvPW/zXV/7rR/Yqw0LDMlpDh4SO3Cdk64H5Go4wUf42BiTybcZOlsTBwt2Nl32BM0h+f0eQrqTRV1JBNdO2aKNckKQxsQxwGWn9yvdZFGiIaX+FGQfqEPjejQn/vGJLd+sjjQyBrFxtY0xlMuQ3r5iexbvg4huUrcYJy04lFx6o/2cmEEpGgHgtU0ROg/3EMLiIIl0WiXuWo3GnPN49QGpbClPbH5U62fmlb+WIoxDYTpjvRbDevknjNMYIYc4Dvkha5QU22Xz/0onTEyQtVlKLjRyEv5uwFZw8h2VbXnvAuYKJ1dViFwQ5uWNetgsPaOEvVolTnBTQORvlNq1GD7SssAC4p4R/LTVW17gpeJ7pYCTynTjCkB7icp882IgAZhd0GeAH4nj+wtMFAMya+6pVQ== acme@le-auth"
```

Für das Vorbereiten der Hosts werden folgende Variablen benötigt:

* `le.user`: Name des Users, welcher für die Zertifikatsverwaltung angelegt werden soll. Dort werden die Konfigurationstemplates gerendert und die Zertifikate samt Private Key abgelegt. Sein Homeverzeichnis liegt in /etc/ssl/`le.user` 
* `le.host`: Hier wird die Host-IP definiert, von dem sich der User mit dem SSH-Key anmelden darf. Erwartet wird eine IP-Addresse (IPv4 oder v6) mit der CIDR-Notation. 
* `le.key`: Der SSH-Key, welcher zur Anmeldung verwendet werden soll. Idealerweise ohne Passphrase (Wie in der Vorbereitung generiert)

##### letsencrypt-host-renew-nginx

Hier sind ebenfalls einige Variablen zu definieren:

```
acme@le-auth:~/ansible-roles$ cat host-renew.yml
---
- hosts: webservers
  roles:
    - letsencrypt-host-renew-nginx
  vars:
    le:
      user: bcssl
      cert: /home/acme/.acme.sh/biocrafting.net/fullchain.cer
      key: /home/acme/.acme.sh/biocrafting.net/biocrafting.net.key
```
* `le.user`: Name des Users, welche in der Rolle `letsencrypt-prepare-nginx` angelegt wurde
* `le.cert`: Dateipfad des Zertifikates
* `le.key`: Dateipfad des Zertifikat-Schlüssel. Dieser wird nach der ersten Domain benannt, für welche das Zertifikat gilt. In meinem Beispiel weiter unten also `biocrafting.net`


#### Script zum generieren des Zertifikates

Ich habe ein kleines Script geschrieben, welches acme.sh und Ansible mit den richtigen Parameter ausführt. Dies wird dann per Cronjob am Monatsanfang ausgeführt.
Hier werden auch die Domains angegeben.

Falls ein vordefinierter Key benutzt wird, kann man den Ordner, in dem das Zertifikat abgespeichert wird, einfach manuell anlegen und den Key mit dem korrekten Dateinamen hinterlegen.


vim ~/acme_bc.sh
```
#!/bin/bash
Domains=(biocrafting.net www.biocrafting.net)
for i in "${Domains[@]}"
do
   :
    domain_list+="-d $i "
done

#echo $domain_list
/home/acme/.acme.sh/acme.sh --force --issue $domain_list -w /var/www/acme-challenges/

if [ $? -eq 0 ]
then
  ansible-playbook -u bcssl ansible-roles/host-renew.yml
  exit 0
else
  echo "Failure"
  exit 1
fi

```

Entsprechende Cronjob Konfiguration:

```
MAILTO=example@example.com
15 0 6 * * /home/acme/acme_bc.sh
```

Ich bekomme durch diese Konfiguration eine Mail beim Erfolg/Misserfolg.
Bitte beachtet, dass das Script beim Fehlschlag von acme.sh nichts auf den Webservern verändert. Dort ist noch manuelles Eingreifen notwendig, da ich aber quasi 2 Monate "Puffer" habe, ist dies zu verschmerzen.

## Ergebnis und weiteres Verfahren

Wenn das Script erfolgreich ausgeführt wurde, liegen auf dem Webserver folgende Dateien:

* /etc/ssl/`user`/letsencrypt/nginx-ssl.conf
* /etc/ssl/`user`/letsencrypt/`Jahr-Monat`/cert.pem
* /etc/ssl/`user`/letsencrypt/`Jahr-Monat`/key.pem

Die nginx Konfiguration wird demzufolge jeden Monat neu generiert, da sich der Pfad zu dem Zertifikat ändert.
So kann ich auch manuell auf dem Server überprüfen, ob das Zertfikat wirklich jeden Monat erneuert wurde. Auch besitze ich so eine Art Historie, welche eventuell nützlich werden könnte.

Für die VHosts ändert sich der Pfad der SSL-Konfiguration nicht - sie muss also nur beim erstmaligen Setup konfiguriert werden.

```
server {
    listen  [::]:443 ssl;
    keepalive_timeout   70;
    server_name ipv6.biocrafting.net;
    
    ...

    include /etc/nginx/global/letsencrypt-auth.conf;
    include /etc/ssl/bcssl/letsencrypt/nginx-ssl.conf;
}
```


Hiermit habe ich schon mal die Webserver-Konfiguration abgefackelt.

Was noch fehlt sind die Konfiguration für:

* slapd (LDAP)
* Dovecot (IMAP/POP3)
* Postfix (Mailserver)

An entsprechenden Rollen für die Dienste arbeite ich bereits und werde, wenn diese Spruchreif geworden sind, ebenfalls in dem github-Repository veröffentlichen. Stay tuned!
