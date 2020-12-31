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


## Vorbereitung MySQL-Datenbank

Im folgenden Abschnitt wird das Erstellen einer MySQL-Datenbank beschrieben. Zuerst wird der Server mit `sudo apt install mysql-server` installiert, man sollte jedoch zuvor die neusten Updates mit `sudo apt update` runterladen. Anschließend kann der Dienst mit `sudo mysql` gestartet und mit dem Befehl `CREATE DATABASE my_logs;` eine Datenbank namens my_logs angelegt werden. Innerhalb des MySQL-Dienstes ist es wichtig, Befehle immer mit einem ";" abzuschließen. Nach erstellen kann die Datenbank mit dem Befehl `use  my_logs;` verwendet werden. Nun kann eine Tabelle per Befehl erstellt werden, oder wie in unserem Fall eine zuvor erstellte Textdatei (logTabelle.sql), die den Befehl beinhaltet außerhalb des MySQL-Dienstes mittels `sudo mysql my_logs < logTabelle.sql` eingelesen werden. Somit haben wir eine Tabelle innerhalb unserer Datenbank my_logs erstellt und können diese nun für Einträge nutzen.

### Inhalt der "logTabelle.sql" Datei:
```sql
CREATE TABLE BackupLogs (
    id int NOT NULL AUTO_INCREMENT,
    log_type varchar(255) NOT NULL,
    success int,
    info varchar(255),
    PRIMARY KEY (id)
);
```

## Durchführung

Ich starte zwei Linux 20.04.1 LTS Maschinen über das Virtualisierungsprogramm VirtualBox, welche sich im selben Netztwerk befinden. Anschließend wechsle ich innerhalb der Maschine mit der IP-Adresse 192.168.56.31 in das Verzeichnis, wo sich das Script '( backupScript.bash )' befindet, welches in meinem Fall unter './tinf/Backup/' liegt. Anschließend führe ich das vorher genannte Script mit dem Befehl bash backupScript.bah Dokumente backup aus. Die beiden Argumente werden für die Angabe der Verzeichnisse benötigt. Das Script speichert, nachdem alle Bedingungen erfüllt sind die Backupdatei in den per Argument übergebenen Ordner und überträgt anschließend die Datei an den eingetragenen Rechner mit der IP-Adresse 192.168.56.32 in das hinterlegte Verzeichnis. Zusätzlich wird eine Logdatei erstellt, welche die Fehler mit dokumentiert. Zudem wird ein Eintrag mit dem Namen der Datei (Datumsstempel_Dokumente) in die oben genannte Datenbank erstellt. Die genaue Funktion kann dem Script folgenden entnommen werden:

```bash
#!/bin/bash
#title           :backupScript.bash
#description     :Automatisiertes Backupscript mittels rsync
#author          :mathiasrudig
#date            :20201229
#version         :0.3    
#usage           :bash backupScript.bash Dokumente backup
#bash_version    :5.0.17(1)-release
#===============================================================

##Variablen erzeugen
backupFile=$(date +%Y%m%d%H%M%S)"_Dokumente"             #Dateiname einer Variable zuweisen
folderToBackup=$1                                        #Mitgabe des erstem Argument - Der Ordner aus dem ein Backup erstellt werden soll
backupFolder=$2                                          #Mitgabe des zweitem Argument - Der Ordner in dem das komprimierte Backup gespeichert werden soll
logDatei=logBackup.txt                                   #Dateiname der Logdatei einer Variable zuweisen

echo "=====================================" | tee -a ./${logDatei}
echo $(date +%Y%m%d%H%M%S)" Script wird gestartet" | tee -a ./${logDatei}

##Abfrage, ob die Argumente einen String enthalten und somit gültig sind
if [ -z $1 ] || [ -z $2 ]; then
        echo "Fehler bei den mitgegebenen Argumenten!" | tee -a ./${logDatei}
        echo "Script wurde abgebrochen!" | tee -a ./${logDatei}
        echo "Script wurde nicht ordnungsgemäß ausgeführt, bitte überprüfen Sie die logBackup.txt"
        exit 1
fi

##Abfrage, falls der Ordner folderToBackup nicht existiert, soll das Script beendet werden
if [ ! -d ${folderToBackup} ]; then
        echo "Der Ordner existiert nicht!" | tee -a ./${logDatei}
        echo "Script wurde abgebrochen!" | tee -a ./${logDatei}
        echo "Script wurde nicht ordnungsgemäß ausgeführt, bitte überprüfen Sie die logBackup.txt"
        exit 1
fi

##Abfrage, falls der Ordner backup nicht existiert, sollte einer erstellt werden
if [ ! -d ${backupFolder} ]; then
        echo "Der Ordner ${backupFolder} wurde erstellt!" | tee -a ./${logDatei}
        mkdir ${backupFolder}
fi

##Kompriemierte .tar.gz Datei vom folderToBackup mit Datumsstempel_Dokumente als Dateinamen innerhalb dem backupFolder erstellen
tar -czf ${backupFolder}/${backupFile}.tar.gz ${folderToBackup} 2>> ./${logDatei}
if [ $? -eq 0 ]; then #Abfrage ob beim Erstellen der Datei ein Fehler aufgetreten ist
        echo "Datei wurde erfolgreich erstellt!" | tee -a ./${logDatei}
else
        echo "Datei konnte nicht erstellt werden!" | tee -a ./${logDatei}
        echo "Script wurde nicht ordnungsgemäß ausgeführt, bitte überprüfen Sie die logBackup.txt"
        exit 1
fi

##Datei an den eingetragenen Rechner übertragen und einen Eintrag in die Datebank my_logs erstellen
if [ -s ./${backupFolder}/${backupFile}.tar.gz ]; then
        rsync --numeric-ids -avz --stats ${backupFolder}/${backupFile}.tar.gz mathias@192.168.56.32:/home/mathias/tinf 2>> ./${logDatei} #Dateiübertragung per ssh
        if [ $? -eq 0 ]; then #Abfrage, ob beim Übertragen der Datei ein Fehler aufgetreten ist
                echo "Datei wurde erfolgreich übertragen!" | tee -a ./${logDatei}
                sudo mysql -e "INSERT INTO BackupLogs (\`log_type\`,\`success\`,\`info\`) VALUES ('SYS_BACKUP',1,'$backupFile');" my_logs 2>> ./${logDatei} #Schreiben in eine Datenbank mit success 1
                if [ $? -eq 0 ]; then #Abfrage, ob beim Schreiben in die Datenbank ein Fehler aufgetreten ist
                        echo "Eintrag in Datenbank war erfolgreich!" | tee -a ./${logDatei}
                        echo $(date +%Y%m%d%H%M%S)" Das Sicherungsscript wurde erfolgreich ausgeführt!" | tee -a ./${logDatei}
                else
                        echo "Eintrag in Datenbank konnte nicht durchgeführt werden!" | tee -a ./${logDatei}
                        echo "Script wurde nicht ordnungsgemäß ausgeführt, bitte überprüfen Sie die logBackup.txt"
                        exit 1
                fi
        else
                echo "Datei konnte nicht übertragen werden!" | tee -a ./${logDatei}
                sudo mysql -e "INSERT INTO BackupLogs (\`log_type\`,\`success\`,\`info\`) VALUES ('SYS_BACKUP',0,'$backupFile');" my_logs 2>> ./${logDatei} #Schreiben in eine Datenbank mit success 0
                if [ $? -eq 0 ]; then #Abfrage, ob beim Schreiben in die Datenbank ein Fehler aufgetreten ist
                        echo "Eintrag in Datenbank war erfolgreich!" | tee -a ./${logDatei}
                else
                        echo "Eintrag in Datenbank konnte nicht durchgeführt werden!" | tee -a ./${logDatei}
                echo "Script wurde nicht ordnungsgemäß ausgeführt, bitte überprüfen Sie die logBackup.txt"
                exit 1
                fi
        fi
        
else
        echo "Datei existiert nicht, oder ist kleiner 0 Byte!" | tee -a ./${logDatei}
        echo "Script wurde abgebrochen!" | tee -a ./${logDatei}
        sudo mysql -e "INSERT INTO BackupLogs (\`log_type\`,\`success\`,\`info\`) VALUES ('SYS_BACKUP',0,'$backupFile');" my_logs 2>> ./${logDatei} #Schreiben in eine Datenbank mit success 0
        if [ $? -eq 0 ]; then #Abfrage, ob beim Schreiben in die Datenbank ein Fehler aufgetreten ist
                        echo "Eintrag in Datenbank war erfolgreich!" | tee -a ./${logDatei}
                else
                        echo "Eintrag in Datenbank konnte nicht durchgeführt werden!" | tee -a ./${logDatei}
                fi
        echo "Script wurde nicht ordnungsgemäß ausgeführt, bitte überprüfen Sie die logBackup.txt"
        exit 1
fi

# ©mathias rudig
```
## Erweiterung mittels "crontab":
Man könnte das Script noch voll mittels crontab Automatisieren, um automatisch einmal pro Tag das Script auszuführen. Dazu muss ein Eintrag unter `crontab -e` in folgender Form angefügt werden:  
```bash
# m h  dom mon dow   command
0 0 * * * /usr/bin/bash /home/mathias/tinf/Backup/backupScript.bash /home/mathias/tinf/Backup/Dokumente /home/mathias/tinf/Backup/backup
```
Wichtig ist, dass man absolute Dateipfade verwendet, damit immer auf die richtigen Verzeichnisse und Dateien zugegriffen wird. Damit das Script nun Auomatisiert läuft, muss noch das Passwort mitgegeben werden, welches nicht als Klartext innerhalb des Scripts eingetragen werden sollte. Da mir zum heutigen Stand noch das nötige Wissen fehlt, wird dies nachgereicht.

## Literaturverzeichnis

* https://wiki.ubuntuusers.de/ - wurde für die Recherche genutzt
* `man rsync` - Manual-Page innerhalb von Linux von rsync
* `man tar` - Manual-Page innerhalb von Linux von tar


                
