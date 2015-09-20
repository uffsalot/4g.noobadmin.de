---
categories: []
date: 2014-05-09T15:51:08+02:00
description: ""
keywords: []
title: .htpasswd schnell ohne apache2-utils anlegen
---

Wenn ihr schnell eine htpasswd Datei erstellen müsst,
aber z. B. nginx benutzt und das apache2-utils Paket nicht installieren wollt,
gibt es mit OpenSSL eine einfache Möglichkeit die Passwörter zu verschlüsseln.
Dies ist sogar sicherer als die Passwörter unverschlüsselt durchs Netz zu blasen 
und bei ominösen Online-Generatoren einzugeben.

Dies erzeugt eine .htpasswd Datei mit dem Benutzer "USER" und den verschlüsselten Passwort "PASSWORD" (Standard Unix password algorithm, begrenzt auf 8 Zeichen)

Neue Benutzer können beliebig angehängt werden.

```
printf "USER:$(openssl passwd -crypt PASSWORD)\n" >> .htpasswd
```

Wer gerne längere Passwörter benutzt sollte sich die Option -1 mal ansehen (MD5-based password algorithm)

```
printf "USER:$(openssl passwd -1 PASSWORD)\n" >> .htpasswd
```