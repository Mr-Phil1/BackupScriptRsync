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
