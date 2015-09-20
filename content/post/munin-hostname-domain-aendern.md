---
categories: []
date: 2014-10-18T00:49:00+02:00
description: ""
keywords: []
title: Munin Hostname/Domain ändern
---

Wenn man gezwungen ist den Hostname eines Munin-Nodes anzupassen ohne die gesammelten Daten zu verlieren, 
steht man vor ein Problem: Munin erkennt nach der Änderung des Hostnames die alten Daten nicht mehr.

Man muss die Round-Robin-Datenbanken umbenennen damit Munin sie zu den neuen alten Host zuordnet. 

___Wichtig___: Vorher unbedingt ein Backup ziehen & Munin vorübergehend deaktivieren! 
Falls man zu langsam arbeitet oder ein ungünstigen Zeitpunkt wählt (Munin schreibt die Datenbanken alle 5 Minuten neu)
kann man sich ganz leicht alle Datensätze des Hosts zerschießen.

 

### Domain ändern:
Ist-Zustand:
nic.teamtt.de

Soll-Zustand:
nic.biocrafting.net

in das Munin Verzeichnis wechseln:

```
cd /var/lib/munin/teamtt.de
```

Und die Dateien umbenennen:

```
rename -v 's/teamtt.de/biocrafting.net/' nic.*
```

Den übergeordneten Ordner muss man auch noch umbenennen:

```
mv /var/lib/munin/teamtt.de /var/lib/munin/biocrafting.net
``` 

### Hostname ändern:
Funktioniert analog zur Änderung der Domain.

Ist-Zustand:
bananapi.brand

Soll-Zustand:
pi.brand
 

in das Munin Verzeichnis wechseln:

```
cd /var/lib/munin/brand
```

Und die Dateien umbenennen:

```
rename -v 's/bananapi/pi/' bananapi.*
``` 

Jetzt darf nicht vergessen werden in der munin.conf die Hosts auch entsprechend abzuändern. Gegebenfalls muss auch in der munin-node.conf des Nodes auch der Hostname ("host_name" Direktive) geändert werden!

Nachdem der Munin Daemon wieder gestartet wurde sollte nach einer kurzen Wartezeit (~5min) sich die ersten Graphen aktualisieren. Der Weekly Graph dagegen wird nur alle 15 Minuten aktualisiert.