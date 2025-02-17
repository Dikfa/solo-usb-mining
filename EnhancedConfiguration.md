# ⛏ Konfigurationstipps

Diese Seite beschreibt mögliche Konfigurationen von CGMIner auf einem Raspbery Pi

## CGMiner in debug mode 

Um möglichst detaillierte Informationen vor allem beim erstmaligen Setup zu bekommen, bietet es sich an CGMiner wie folgt zu starten:

```console
sudo ./cgminer -o stratum+tcp://solo.ckpool.org:3333 -u <BITCOINADDRESS>.<OPTIONAL_NAME> -p x --gekko-compacf-freq 500 --gekko-start-freq 200 --gekko-mine2 --gekko-tune2 60
```

In diesem Modus kann man übersichtlich die Leistung der angeschlossenen Miner beobachten, das Hotplugging-Verhalten untersuchen aber auch einfach nur Einstellungen on-the-fly vornehmen.

---

## CGMiner im Hintergrund...

### ...mit `nohup &`

Der Betrieb im Hintergrund ist notwendig, wenn man nicht permanent eine remote Verbindung via SSH offen halten kann oder will, z.B. beim Betrieb von CGMiner auf einem Raspberry Pi. Der cgminer-Dienst kann einfach mit `nohup` im Hintergrund gestartet werden:

```console
nohup sudo ./cgminer --compact --real-quiet -o stratum+tcp://solo.ckpool.org:3333 -u <BITCOINADDRESS>.<OPTIONAL_NAME> -p x --gekko-compacf-freq 500 --gekko-start-freq 200 --gekko-mine2 --gekko-tune2 60 &
```

Man beachte hier die Optionen `--compact` und `--real-quiet`. Diese Optionen verringern das zu loggende Datenvolumen auf ein Minimum.

Man beachte hierbei das abschliessende `&`. Der Prozess läuft nun im Hintergrund und leitet seine Ausgabe in die Datei `/home/admin/nohup.out` um. Diese kann mit `cat` in der Konsole ausgegeben werden:

```console
cat /home/admin/nohup.out
```

Um den Prozess zu beenden, muss die Prozess-ID mittels `kill` terminiert werden. Dazu sucht man zuerst die Prozess-ID:

```console
ps aux | grep cgminer
```

Dies zeigt entsprechende Prozess-IDs von cgminer an und können wie folgt beendet werden:

```console
sudo kill <PROZESSID>
```

---

### ...mit `screen` (der elegantere Ansatz)

zuerst muss screen installiert werden, wie zuvor auch mittels apt-get:

```console
sudo apt-get update

sudo apt-get upgrade -y

sudo apt-get install screen
```

Zuerst generieren wir ein ausführbares Shellskript zum Start des Miners mit einem Editor wie nano (`<USER>` mit namen des erzeugenden Users ersetzen):

```console
sudo nano -w /home/<USER>/cgminer/cgminer.sh
```

Der Inhalt des Skriptes ist der Startbefehl für die Mining-Software cgminer (`<USER>` einsetzen):

```console
#!/bin/bash
cd /home/<USER>/cgminer

sudo ./cgminer -c /root/.cgminer/cgminer.conf 2> "run-`date +%Y%m%d%H%M%S`.log"
```

Hier wurde der gekürzte Startbefehl für cgminer verwendet, die Konfiguration wird aus der Konfigurationsdatei `cgminer.conf` gelesen. Der Befehl `2> "run-'date +%Y%m%d%H%M%S'.log"` lenkt die Ausgabe, wenn der Prozess mit screen im Hintergrund läuft in eine log-Datei um, welche sich auch im cgminer-Verzeichnis befindet.

Nun muss nur noch das Skript ausführbar gemacht werden (`<USER>` einsetzen):

```console
sudo chmod +x /home/<USER>/cgminer/cgminer.sh
```

Um den Mining-Prozess im Hintergrund zu starten und somit auch am Laufen zu halten wenn die SSH-Session beendet wird, rufen wir nun das Shellskript mit dem Startbefehl des Miners mittels `screen` auf (`<USER>` einsetzen):

```console
sudo screen -dm -S miner /home/<USER>/cgminer/cgminer.sh
```

Zur Überprüfung des im Hintergrund nun laufenden Screens kann folgender Befehl verwendet werden:

```console
sudo screen -ls
```

Der Screen kann auch in den Vordergrund gebracht und angezeigt werden:

```console
sudo screen -r miner
```

Die Angabe von `miner` ist bei einem screen nicht notwendig.

Mittels `<CTRL><A>` und anschließendem `<CTRL><D>` kann der Prozess wieder in den Hintergrund gebracht werden. 

---

### Automatischer Start des Miners nach einem Reboot 

Damit die Mining-Software nach jedem Neustart automatisch wieder anläuft, können wir den obigen screen-Befehl auch in die `rc.local` mit aufnehmen. Diese editieren wir mit nano:

```console
sudo nano -w /etc/rc.local
``` 

Vor dem `Exit 0` fügen wir folgende Codezeile ein (`<USER>` einsetzen):

```console
sudo su - -c "screen -dm -S miner /home/<USER>/cgminer/cgminer.sh"
```

Mittels `-c` wird dem Superuser ein Kommando mitgegeben, in unserem Fall der Aufruf des Miners via Shellskript. Die `rc.local` sieht dann aus wie folgt (`<USER>` einsetzen):

```console
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
fi

sudo su - -c "screen -dm -S miner /home/<USER>/cgminer/cgminer.sh"

exit 0
```

Zur Überprüfung nun lediglich neustarten, im Falle von Raspiblitz unbedingt per Software-Befehl, so dass bitcoind und lnd sauber gestoppt werden: 

 ```console
restart
```

Ob es funktioniert hat kann wieder mittels `sudo screen -r` geprüft werden.

---

## cgminer mit mehreren Pools betreiben

Es bietet sich an auf mehreren Hochzeiten zu tanzen, um z.B. 70% der Zeit in einen Pool zu minen (Solo-Pool-Mining), 30% aber auf eigene Faust zu minen (echtes Solomining). Dies hat den Vorteil, dass man im Falle eines Blockfundes des Pools seinen Beitrag bekommt, also entweder den Anteil der geleisteten Shares wenn jemand anderes einen Block findet ODER die vereinbarte Blockbelohnung des Solo-Mining-Pools wenn man den Block selbst findet, gleichzeitig aber auch sein Glück ausreizt um ganz alleine einen Block zu finden. Das muss jeder für sich selbst herausfinden, die Anleitung dazu gibt es trotzdem hier.

Anpassen der Konfigurationsdatei `/root/.cgminer/cgminer.conf` um die Pools und Quotas bekannt zu geben:

```console
sudo nano -w /root/.cgminer/cgminer.conf
```

Eintragen der Pools ähnlich meinem Beispiel:

```console
{
"pools" : [
        {
                "quota" : "70;stratum+tcp://solo.ckpool.org:3333",
                "user" : "<BITCOINADDRESS>.<OPTIONAL_NAME>",
                "pass" : "x"
        },
        {
                "quota" : "30;stratum+tcp://solo.ckpool.org:3333",
                "user" : "<BITCOINADDRESS>.<OPTIONAL_NAME>",
                "pass" : "x"
        }
]
,
...
"load-balance" : true
...
}
```

Wichtig hierbei die quotas (in meinem Beispiel 70% und 30%), aber auch die Option `load-balance` auf `true` zu setzen.

Das Verwenden der Config-Datei hat den weiteren Charme, dass man nicht mehr sämtliche Parameter an den Miner übergeben muss, es reicht lediglich `sudo ./cgminer` aufzurufen und die weitere Parametrierung findet über die onfigurationsdatei statt. Durch die Option `sudo ./cgminer -c /root/.cgminer/cgminer.conf` önnen außerdem unterschiedliche Konfigurationen geladen werden.

---

####  [⛏ Mining Software - GUI Konfiguration](cgminer_GUIConfiguration.md)  ᐊ  previous | next  ᐅ  [⛏ Miner Einstellungen MHz/ mV](miner-settings.md)
