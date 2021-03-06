firmware
========

# Was ist Freifunk?
Freifunk ist eine nicht-kommerzielle Initiative für freie Funknetzwerke. Jeder Nutzer im Freifunk-Netz stellt einen günstigen WLAN-Router für sich selbst und den Datentransfer der anderen Teilnehmer zur Verfügung. Dieses Netzwerk kann von jedem genutzt werden.

Weitere Informationen gibt es auf <https://freifunk.net/> und auf <https://wiki.freifunk-franken.de/w/Hauptseite>.

# Firmware selbst kompilieren
## Voraussetzungen
* `apt-get install zlib1g-dev lua5.2 build-essential unzip libncurses-dev gawk git subversion realpath libssl-dev` (Sicherlich müssen noch mehr Abhängigkeiten installiert werden, diese Liste wird sich hoffentlich nach und nach füllen. Ein erster Ansatzpunkt sind die Abhängigkeiten von OpenWrt selbst)
* `git clone https://github.com/FreifunkFranken/firmware.git`
* `cd firmware`

## Erste Schritte
Je nachdem, für welche Hardware die Firmware gebaut werden soll, muss das BSP gewählt werden:

* `./buildscript selectbsp bsp/board_ar71xx.bsp`
* Um die vorhandenen BSPs zu sehen, kann `./buildscript selectbsp help` ausgeführt werden.

## Was ist ein BSP?
Ein BSP (Board-Support-Package) beschreibt, was zu tun ist, damit ein Firmware Image für eine spezielle Hardware gebaut werden kann.
Typischerweise ist eine Ordner-Struktur wie folgt vorhanden:
* .config
* root_file_system/
  * etc/
    * rc.local.board
    * config/
      * board
      * network
      * system
    * crontabs/
      * root

Die Daten des BSP werden nie alleine verwendet. Zuerst werden immer die Daten aus dem "default"-BSP zum Ziel kopiert, erst danach werden die Daten des eigentlichen BSPs dazu kopiert. Durch diesen Effekt kann ein BSP die "default" Daten überschreiben.

## Die Verwendung des Buildscripts
Die BSP-Datei wird durch das Buildscript automatisch als dot-Script geladen, somit stehen dort alle Funktionen zur Verfügung.
Das Buildscript generiert ein dynamisches sed-Script. Dies geschieht, damit die Templates mit den richtigen Werten gefüllt werden können.

### `./buildscript prepare`
* Sourcen werden in einen separaten src-Folder geladen, sofern diese nicht schon da sind. Zu den Sourcen zählen folgende Komponenten:
  * OpenWrt
  * Sämtliche Packages (ggf. werden Patches angewandt)

* Ein ggf. altes Target wird gelöscht
* OpenWrt wird ins Target exportiert (kopiert)
* Eine OpenWrt Feed-Config wird mit dem lokalen Source Verzeichnis als Quelle angelegt
* Die Feeds werden geladen
* Spezielle Auswahl an Paketen wird geladen
* Patches werden angewandt
* board_prepare() aus dem BSP wird aufgerufen (wird z.B. für Patches für eine bestimmte Hardware verwendet)

### `./buildscript config openwrt`
Um das Arbeiten mit den .config-Dateien von OpenWrt zu vereinfachen, bietet das Buildscript die Möglichkeit das `menuconfig` von OpenWrt aufzurufen. Nachdem man die gewünschten Einstellungen vorgenommen hat, hat man die Möglichkeit, die frisch editierte Konfiguration in das BSP zu übernehmen.
Dieses Kommando arbeitet folgendermaßen:
* prebuild
* OpenWrt: `make menuconfig`
* Speichern, y/n?
* Config-Format vereinfachen
* Config ins BSP zurück speichern

### `./buildscript build`
* prebuild
  * $target/files aufräumen
    * (In $target/files liegen Dateien, die später direkt im Ziel-Image landen)
  * Files aus default-bsp und bsp kopieren
  * OpenWrt- und Kernel-Config kopieren
  * board_prebuild() aus dem BSP wird aufgerufen
* Templates transformieren
* GIT Versionen speichern: $target/files/etc/firmware_release
* OpenWrt: make
* postbuild
  * board_postbuild() wird aufgerufen

## Erweiterung eines BSPs
Beispielhaftes Vorgehen um den WR1043V2 zu unterstützen.

### Repository auschecken
```
git clone https://github.com/FreifunkFranken/firmware.git
cd firmware
```

### Erste Images erzeugen
Du fügst die Dateinamen der Images, die zusätzlich kopiert werden sollen, in das `images`-Array ein:

```
vim bsp/board_ar71xx.bsp
images=(
    // ...
    openwrt-${chipset}-${subtarget}-tl-wr1043nd-v2-squashfs-sysupgrade.bin"
    openwrt-${chipset}-${subtarget}-tl-wr1043nd-v2-squashfs-factory.bin"
    // ...
)
```

Dann muss auf jeden Fall noch das Netzwerk richtig konfiguriert werden. Dazu muss man den Router sehr gut kennen, i.d.R. lernt man den erst beim Verwenden kennen, daher ist ein guter Startpunkt die Config vom v1 zu kopieren und erstmal zu gucken was passiert:
```
cp bsp/wr1043nd/root_file_system/etc/network.tl-wr1043nd-v1 bsp/wr1043nd/root_file_system/etc/network.tl-wr1043nd-v2
```
Anschließend kann ein erstes Image erzeugt werden:
```
./buildscript selectbsp bsp/board_wr1043nd.bsp

./buildscript prepare
./buildscript build
```
Jetzt gehst du n Kaffee trinken.

**Vorsicht:** Für das ar71xx BSP wurde das tiny-Subtarget so erweitert, so dass auch die Geräte aus dem OpenWRT generic Subtarget damit gebaut werden können. (vgl. [Patch](build_patches/openwrt/0005-allow-building-all-devives-as-tiny.patch))

Soll das ar71xx BSP um ein Gerät erweitert werden, das bei OpenWRT unter generic geführt wird, muss die Kernel Konfiguration entsprechend um die passende MIPS MACH erweitert werden. Ansonsten führt das resultierende Image zu einem Bootloop, wenn es installiert wird.

### Netzwerkeinstellungen korrekt setzen
Am Ende sollte im bin/ Verzeichnis das Image für v1 und v2 liegen. Das v2 Image wird auf den Router geflasht. Achtung: Eventuell ist das Netzwerk jetzt so falsch eingestellt, dass man nicht mehr über Netzwerk auf den Router zugreifen kann. Am einfachsten ist es den Router dann über eine serielle Konsole zu verwenden. Theoretisch kann man an den unterschiedlichen LAN-Ports mit der IPv6 Link-Local aus der MAC Adresse des Geräts versuchen drauf zu kommen. Es kann auch sein, dass die IPv6 +/- 1 am Ende hat. Letztlich kann das funktionieren, ist aber aufwändig und da am LAN Einstellungen verändert werden sollen, ist die serielle Konsole das Mittel der Wahl!
Wenn man dann auf dem Router drauf ist, muss als erstes festgestellt werden, welches Ethernet-Device für den WAN Port zuständig ist. Mir sind da folgende Möglichkeiten bekannt. a) WAN ist eth0, b) WAN ist eth1, c) WAN ist teil vom Switch eth0. Dementsprechend wird das WANDEV auf dem Router in der /etc/network.tl-wr1043nd-v2 konfiguriert. Wenn WAN ein eigenes ethX hat, dann muss WAN_PORTS="" sein. Dann muss eingestellt werden welches Ethernet-Device an dem internen Switch angeschlossen ist (swconfig list). Dieses wird als SWITCHDEV konfiguriert. Es muss noch eingestellt werden, welches Ethernet oder Wifi Device die MAC Adresse hat, die auch unter dem Gerät steht. Dieses Device wird als ROUTERMAC eingetragen. Nun ist es an der Zeit die Einstellungen zu testen, dafür muss die falsche Netzwerk-Config zurück gesetzt werden:
```
cp /rom/etc/config/network /etc/config/network
reboot
```

### Switch konfigurieren
Wenn der Router wieder hochgefahren ist, sollten die Einstellungen sein, so wie sie konfiguriert wurden. Gegebenenfalls muss man hier noch mal eine Runde drehen, wenn etwas nicht richtig war. Ansonsten ist es jetzt an der Zeit den Switch einzustellen. Das geht am einfachsten, wenn man die Einstellungen nun direkt in der /etc/config/network vornimmt. Dabei ist eth0_2 der WAN Port (sofern er über den Switch läuft). eth0_1 sind die Client-Ports und eth0_3 sind die Ports um Batman Knoten zu verbinden. Am Anfang weiß man meist noch nicht welcher Switch Port wirklich am Router wo rausgeführt ist. Manchmal kann es helfen einen Port nach dem anderen aktiv zu schalten (Rechner anstecken) und die Ausgabe von swconfig anzugucken (z.B. `swconfig dev switch0 show`). Das ganze ist manchmal ein wenig try-and-error. :( Aber wenn man denkt es passt, prüft man alles durch. Tauchen BATMAN Nachbarn in `batctl o` auf und aktualisiert sich die Anzeige, wenn ein anderer Knoten an dem Batman-Port angeschlossen wird? Funktioniert das mit beiden Ports? Taucht ein PC in `/etc/showmacs.sh br-mesh` auf, den man an die Client-Ports angeschlossen hat? Wenn am Ende alles passt, übernimmt man die Switch-Config in die /etc/network.tl-wr1043nd-v2 und probiert das ganze nochmal aus:
```
cp /rom/etc/config/network /etc/config/network
reboot
```

### Einstellungen testen und ins BSP übernehmen
Wenn jetzt die Ports immer noch alle korrekt funktionieren kann man die Datei auf den eigenen PC kopieren:
```
scp root@[ipv6ll%scope]:/etc/network.tl-wr1043nd-v2 /path/to/git/firmware/bsp/wr1043nd/root_file_system/etc/network.tl-wr1043nd-v2
```

### BSP commiten und Patch erzeugen
Nun kann man mit `git status` die Änderungen sehen. Mit `git add` staged man diese und mit `git commit` checkt man sie ein. `git format-patch origin/HEAD` erzeugt dann aus deinen Commits ein (oder mehr) Patches. Diese schickst du dann mit `git send-email --to franken-dev@freifunk.net *.patch` an unsere Liste. Dort nimmt sich jemand die Zeit und schaut kurz drüber und wenn alles passt finden deine Änderungen in den Hauptentwicklungszweig und sind ab dann Teil der Freifunk-Franken-Firmware.

### Patch schicken
Auf der Mailingliste franken-dev@freifunk.net kannst du natürlich jederzeit Fragen stellen, falls etwas nicht klar sein sollte.

## Hinzufügen von Paketen zum Image
Das Hinzufügen von Paketen sollte mit Bedacht erfolgen, da dies (bei unvorsichtiger Konfiguration) den Betrieb des Routers und eventuell des Freifunk-Netzes beeinträchtigen könnte.
Mit dem Firmware-Verzeichnis als Arbeitsverzeichnis kann mittels des Befehls `./build/<target>/scripts/feeds install <paket>` ein Paket zur menuconfig hinzugefügt werden.
Mittels des schon bekannten `./buildscript config openwrt` kann das Paket dann ausgewählt werden. Es wird beim anschließenden Build zum Image hinzugefügt.
