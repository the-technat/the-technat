---
title: "minecraft_server"
---

## Einleitung

Diese Anleitung zeigt wie der offiziele Minecraft Server (Java-Edition) von Mojang sowohl auf Windows als auch auf Linux installiert wird.

## Vorbedingungen

Die folgenden Annahmen / Rahmenbedingungen werden getroffen / angenommen für diese Anleitung:

-   Ein Server (Windows/Linux) wurde bereits erstellt und man kann mittels SSH/RDP darauf verbinden
-   Der Server hat eine öffentliche IP Adresse oder Port 25565 TCP/UDP wurde mittels Port-Forwarding freigeschaltet
-   Der Server ist pingbar im Internet (ICMP traffic erlaubt)
-   Der Server ist aktualisiert und gepatched

## Windows

### Java

Der Minecraft Server ist in Java geschrieben. Das bedeutet das wir das Java Runtime Environment installieren müssen. Dieses kannst du [hier](https://www.java.com/de/download/) herunterladen, doppelklicken und installieren. 

Zusätzlich zum JRE benötigen die neueren Versionen des Minecraft Server neu auch das JDK (Java Development Kit). Siehe [hier](https://www.sportskeeda.com/minecraft/how-fix-jni-error-java-edition-setting-minecraft-server) für eine Erklärung wieso das so ist. Herunterladen kannst du das JDK [hier](https://www.oracle.com/java/technologies/downloads/#jdk18-windows) (x64 Installer). Mit einem doppelklick installierst du das JDK auf dem Server.

Danach solltest du den Server kurz neu starten.

Sobald du wieder mit dem Server verbunden bist, öffne mittels Windows+R ein `cmd.exe` und prüfe welche Java Version installiert ist:

![](/java-version.png)

Wenn du einen Output bekommst, weisst du das Java korrekt installiert wurde. Wenn es etwas heisst von "Java executable konnte nicht gefunden werden", stimmt etwas mit der Java-Installation noch nicht und du musst nochmals über die Bücher.

### Minecraft Server download

Sobald java installiert ist, können wir uns den Minecraft Server herunterladen. Doch zuerst erstelle dir einen Ordner im Pfad `C:\Program Files\MinecraftServer`. Danach gehst du auf [diese Website](http://www.minecraft.net/de-de/download/server) und lädst dir die `.jar` Datei herunter und verschiebst sie in den vorher erstellen Ordner.


Beachte: Du lädst hier eine `.jar` Datei direkt herunter, der Windows Defener oder dein Browser werden dir sehr wahrscheinlich melden das diese Datei potenziel schädlich ist für deinen Computer. Diese Warnung kannst du getrost ignorieren.

Im Anschluss sollte das ganze so aussehen:

![](/server_download.png)

### Minecraft Server eula

Sobald die .jar Datei dort ist wo sie sein sollte, kannst du sie mit einem Doppelklick starten. Dabei werden verschiedene Dateien und Ordner im aktuellen Verzeichnis erstellt und der Server wieder beendet. Nachher sieht dein Minecraft Ordner so aus:

![](/server_initialisieren.png)

Du solltest nun eine Datei namens `eula.txt` im aktuellen Verzeichnis finden. Diese musst du öffnen (am besten mit Notepad) und den Wert von `false` auf `true` setzten:

![](/eula.png)

Speichere das ganze wieder und schliesse Notepad.

### Minecraft Server starten

Nun ist mehr oder weniger alles soweit, dass der Minecraft Server gestartet werden kann. Damit wir beim Start des Servers gewisse Optionen mitgeben können, schreiben wir einen "Wrapper" für das `.jar` in Form einer `.bat` Datei.

Öffne Notepad, erstelle eine neue Datei und kopiere folgende Zeile hinein:

![](/start_bat.png)

Beachte: Die 1024M in diesem Befehl sagen Java wie viel Memory Java haben darf. Je nachdem wie viel Memory dein Server hat kannst du Java mehr als `1024M` geben. Mein Server hat 16GB Memory und ich habe Java `8192M` Memory zugewiesen. Dies hilft, damit auch mehrere Spieler flüssig gleichzeitig spielen können.

Danach speichere die Datei ab: Gib ihr den namen `start.bat`. Unter dem Dateinamen gibst du als Typ "Alle Dateien" an und die Codierung sollte `ANSI` sein:

![](/start_bat_save_as.png)

Danach kannst du die Datei `start.bat` mit einem Doppelklick ausführen. Dadurch wird der Minecraft Server gestartet, es öffnet sich ein CMD fenster und der Server fängt an seine Welt aufzubauen:

![](/server_spawn_area.png)

### Windows Firewall öffnen

Als letztes müssen wir noch die Windows Firewall des Servers öffnen, damit Port 25565 TCP/UDP hineingelassen wird.

Suche in der Windows Suche nach "firewall with" und öffne den ersten Treffer:

![](/search_firewall.png)

 Links in der Spalte sollte es irgendwo ein Reiter "Inbound Rules" haben. Klicke darauf, damit sich die Übersicht mit allen Regeln öffnet:

![](/inbound_rules.png)

Wenn du da bist, machst du einen Rechtsklick auf “Inbound Rules” und wählst “New Rule …”. Im sich öffnenden Dialog gibst du dann die folgenden Werte ein:

Rule Type: Port

Protocol and Ports: TCP, 25565
Action: Allow the connection

Profile: Alles angekreuzt
Name: "Minecraft Server (TCP)"

Die gleiche Rule erstellt du ein zweites mal, diesmal aber für UDP, 25565.

Die Rule sieht dann etwa so aus:

![](/minecraft_fw_rule.png)

### Testen

Nun ist es soweit! Öffne Minecraft auf deinem Laptop, wähle Multiplayer und füge den Server zu deiner Liste hinzu.

## Linux

Da du nun bereits den Minecraft Server auf Windows aufgesetzt hast, können wir einen Schritt weiter gehen. 

Linux ist wie Windows ein Betriebssystem, welches man mit und auch ohne einer grafischen Benutzeroberfläche wählen kann. 

Wie du es bereits ahnen kannst, werden wir dies nun ohne Benutzeroberfläche tun. Wie das geht? Wirst du gleich sehen!

![(smile)](https://wikit.post.ch/s/mrdsxn/8703/51k4y0/_/images/icons/emoticons/smile.svg)

### VM auf den neusten Stand bringen

```plaintext
sudo apt update && sudo apt upgrade -y
```

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_9-31-56.png?version=1&modificationDate=1655537517000&api=v2)

### Java installieren

Wie bei Windows mussen wir auch hier Java installieren:

```plaintext
sudo apt install openjdk-17-jre-headless -y
```

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_9-38-25.png?version=1&modificationDate=1655537905000&api=v2)

Um sicherzustellen, dass Java installier ist, kannst du versuchen die Version von Java herauszugeben:

```plaintext
java -version
```

Dies sollte in etwa so aussehen:

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_9-39-56.png?version=1&modificationDate=1655537996000&api=v2)

### Benutzer “Minecraft” erstellen

Damit wir einen eigenen Benutzer für unser spiel haben, erstellen wir doch gleich einen:

```plaintext
sudo useradd -r -m -U -d /opt/minecraft -s /bin/bash minecraft
```

Merke dir, bei Linux heisst es: Keine Antwort ist eine gute Antwort

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_9-50-34.png?version=1&modificationDate=1655538634000&api=v2)

Melde dich nun mit dem Benutzer an:

```plaintext
su - minecraft
```

Wie du siehst, hat sich nun den Benutzer gewechselt:

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_9-52-0.png?version=1&modificationDate=1655538720000&api=v2)

### Minecraft Server herunterladen

Ordner erstellen und diesen öffnen:

```plaintext
mkdir -p ~/server && cd ~/server
```

Nun laden wir uns den Minecraft Server herunter: 

```plaintext
wget https://launcher.mojang.com/v1/objects/e00c4052dac1d59a1188b2aa9d5a87113aaf1122/server.jar
```

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_9-58-37.png?version=1&modificationDate=1655539117000&api=v2)

→ Die neuste Version findest du für in Zukunft immer hier: [https://www.minecraft.net/en-us/download/server](https://www.minecraft.net/en-us/download/server)

Versuchen wir einmal den Server zu starten

```plaintext
java -Xmx1024M -Xms512M -jar server.jar nogui
```

Wie du aber in der untersten Zeile siehst, müssen wir auch hier wieder die EULA aktzeptieren: Mehr Informationen dazu findest du hier: [https://de.wikipedia.org/wiki/Endbenutzer-Lizenzvertrag](https://de.wikipedia.org/wiki/Endbenutzer-Lizenzvertrag)

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_10-4-42.png?version=1&modificationDate=1655539482000&api=v2)

### EULA akzeptieren

Um die EULA zu aktzeptieren, müssen wir folgenden Befehl ausführen: 

```plaintext
sed -i -e 's/false/true/g' eula.txt
```

Damit setzten wir in der Text Datei namens eula.txt den Wert "false" auf "true"

### Server starten

Nun könne wir bereits den Server erneut starten und anfangen zu gamen:

```plaintext
java -Xmx1024M -Xms512M -jar server.jar nogui
```

Doch leider müssen wir jedesmal wenn wir den Server neustarten diesen Befehl selbst ausführen. In der Informatik wollen wir alles automatisch haben, Somit werden wir dies automatisieren!

### Service erstellen

Nun erstellen wir eine neue Datei, welche automatisch beim Start ausgeführt wird. Doch dies darf unser neuer Benutzer nicht tun. Somit melden wir uns wieder mit dem Benutzer ab:

```plaintext
exit
```

Nun erstellen wir die Datei:

```plaintext
nano /etc/systemd/system/minecraft.service
```

Nun hat sich die neue Datei geöffnet und wir können folgendes hinein kopieren:

```plaintext
[Unit]
Description=Minecraft Server
After=network.target

[Service]
User=minecraft
Nice=1
KillMode=none
SuccessExitStatus=0 1
ProtectHome=true
ProtectSystem=full
PrivateDevices=true
NoNewPrivileges=true
WorkingDirectory=/opt/minecraft/server
ExecStart=/usr/bin/java -Xmx1024M -Xms512M -jar server.jar nogui

[Install]
WantedBy=multi-user.target
```

Dies sollte nun so aussehen:

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_10-13-49.png?version=1&modificationDate=1655540029000&api=v2)

Speichere die Datei mit: ctrl + x → y → Entertaste

Lassen den Server die Datei neu finden: 

```plaintext
sudo systemctl daemon-reload
```

Lassen die Datei (service) automatisch starten: 

```plaintext
sudo systemctl enable minecraft
```

Starte die Datei (service) das letzte Mal manuell: 

```plaintext
sudo systemctl start minecraft
```

Schaue, ob der Service Minecraft läuft:

```plaintext
sudo systemctl status minecraft
```

![](https://wikit.post.ch/download/attachments/919612584/image2022-6-18_10-16-20.png?version=1&modificationDate=1655540180000&api=v2)

Wenn hier "active (running)" in grün steht, hast du es geschafft! Gratullation! 

Verbinde dich nun mit dem Server und teste dein Server

## Further Reading

-   [https://www.pcwelt.de/a/so-setzen-sie-ihren-eigenen-minecraft-server-auf,3450161](https://www.pcwelt.de/a/so-setzen-sie-ihren-eigenen-minecraft-server-auf,3450161)
-   [https://minecraft.fandom.com/de/wiki/Anleitungen/Server\_erstellen/Windows](https://minecraft.fandom.com/de/wiki/Anleitungen/Server_erstellen/Windows)
