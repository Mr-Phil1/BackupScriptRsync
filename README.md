# Automatisiertes Linux Bash Backupscript mittels rsync

## Einleitung

Der folgende Bericht beinhaltet ein Bash-Script, welches ein Backup eines Ordners komprimiert erstellen soll und anschließend an einen Remote-Rechner per rsync übertragen soll. Zusätzlich wird noch ein Eintrag mit dem Dateinamen nach der Übertragung in eine MySQL-Datenbank geschrieben. Warum machen wir das? Ich denke Backups, sind in jeglicher Form in der IT-Welt essenziell und unverzichtbar. Datensicherungen können auf vielen Wegen durchgeführt werden, hier beschreibe ich kurz meinen Weg mit dem Programm rsync und einer MySQL-Datenbank.

## Verwendete Technologien

* Oracle Virtual Box version 6.1.16 r140961
* Linux Server version Ubuntu 20.04.1 LTS
* bash version GNU bash, version 5.0.17(1)
* rsync version 3.1.3  protocol version 31
* tar (GNU tar) 1.30
* mysql  Ver 8.0.22-0ubuntu0.20.04.3 for Linux on x86_64 ((Ubuntu))
