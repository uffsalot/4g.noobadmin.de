---
categories: []
date: 2016-08-05T22:12:23+02:00
description: ""
keywords: []
title: Let's Encrypt catchall acme responses
---

...oder: Spaß mit LXC, nginx und ansible

## Vorgeplänkere

Bei [Biocrafting.net](https://www.biocrafting.net) haben wir bisher auf ein Wildcard Zertifikat von Comodo gesetzt.
Allerdings ist dieses vor ein paar Monaten ausgelaufen und angesichts der [Praktiken](https://letsencrypt.org/2016/06/23/defending-our-brand.html) die der Verein an den Tag legt, sehe ich es unter anderem nicht ein, wieder knapp 100€ für das Zertifikat auszugeben.
Da das Zertfikat für 1 Jahr gültig war, habe ich mir nie Gedanken über eine Automatisierung des Vorganges gemacht.
Mit Let's Encrypt sieht es allerdings anders aus. Dort muss für jede Subdomain eine gültige ACME-Challenge präsentiert werden - eine Herausforderung, wenn die knapp 25 Subdomains auf 3 Server verteilt sind. "Erschwerend" kommt hinzu, dass ich auf der Domain HPKP aktiviert habe - Ich kann also nicht mit jedem beliebigen Private Key ein Zertifikat generieren. Eine zentralisierte Lösung musste her.


Daher habe ich mich dazu entschieden einen gesonderten Container zu erzeugen, welcher ausschließlich für die Zertifikatsanforderungen zuständig ist.
So habe ich die komplette Kontrolle über das Deployment und habe auch eine einheitliche SSL-Konfiguration, die vorher auf jedem System unterschiedlich war.

Eckdaten:

* Debian Jessie Container
* Only IPv6 (v4 Adressen sind knapp!)
* Software: [nginx](https://nginx.org), [acme.sh](https://github.com/Neilpang/acme.sh), [Ansible](https://www.ansible.com/)


Ich habe den Container __le-auth__ getauft. Er bekommt auch eine eigene Subdomain, nämlich `le-auth.biocrafting.net`.
Der Container stellt einen Let's Encrypt Client und einen Webserver (Für das Ausliefern der ACME Auth Challenges) bereit.

Natürlich könnt ihr auch direkt mit der IP arbeiten, oder den Server z. B. über Digital Ocean erstellen.

## Vorbereitungen

### Auf den Webservern

Für jede Domain muss ein Reverse-Proxy eingerichtet werden, damit das .well-known/acme-challenges/ Verzeichnis zu dem le-auth.biocrafting.net Server weitergeleitet wird.

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

Benötige Software installieren/DNS Server setzen/Nutzer hinzufügen

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
```

nginx vhost anlegen

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


