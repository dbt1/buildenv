Dieses Skript dient als Werkzeug zur Vereinfachung der Erstellung einer Umgebung für Entwicklung und des Build-Prozesses für Images die mit Neutrino als Benutzeroberfläche auf unterschiedlichen Hardware-Plattformen laufen. Es automatisiert einige Schritte, die für die Einrichtung einer konsistenten und funktionalen Entwicklungs- und Build-Umgebung erforderlich sind, indem es die notwendigen Abhängigkeiten und grundlegende Konfigurationen sowie Meta-Layer  voreinrichtet und benutzerdefinierte Einstellungen ermöglicht. Das Skript zielt darauf ab, eine Grundlage zu bieten, auf der man aufbauen und experimentieren kann, um eigene angepasste Versionen von Tuxbox-Neutrino Images zu erstellen, zu aktualisieren und zu pflegen.

- [1. Vorbereitung](#1-vorbereitung)
	- [1.1 Erforderliche Host-Pakete installieren](#11-erforderliche-host-pakete-installieren)
		- [1.1.1 Empfohlene Zusatzpakete zur grafischen Unterstützung und Analyse](#111-empfohlene-zusatzpakete-zur-grafischen-unterstützung-und-analyse)
	- [1.2 Git vorbereiten (falls erforderlich)](#12-git-vorbereiten-falls-erforderlich)
	- [1.3 Init-Skript klonen](#13-init-skript-klonen)
	- [1.4 Init-Skript ausführen](#14-init-skript-ausführen)
	- [1.5 Struktur der Buildumgebung](#15-struktur-der-buildumgebung)
- [2. Image bauen](#2-image-bauen)
	- [2.1 Box wählen](#21-box-wählen)
	- [2.2 Starte Umgebungsskript](#22-starte-umgebungsskript)
	- [2.3 Image erstellen](#23-image-erstellen)
- [3. Aktualisierung](#3-aktualisierung)
	- [3.1 Image aktualisieren](#31-image-aktualisieren)
	- [3.2 Paket aktualisieren](#32-paket-aktualisieren)
	- [3.3 Meta-Layer-Repositorys aktualisieren](#33-meta-layer-repositorys-aktualisieren)
- [4. Eigene Anpassungen](#4-eigene-anpassungen)
	- [4.1 Konfiguration](#41-konfiguration)
		- [4.1.1 Konfigurationsdateien](#411-konfigurationsdateien)
		- [4.1.2 bblayers.conf](#412-bblayersconf)
		- [4.1.3 Konfiguration zurücksetzen](#413-konfiguration-zurücksetzen)
	- [4.3 Recipes](#43-recipes)
	- [4.4 Pakete](#44-pakete)
		- [4.4.1 Neutrino Quellcode im Workspace bearbeiten (Beispiel)](#441-neutrino-quellcode-im-workspace-bearbeiten-beispiel)
- [5. Neubau eines einzelnen Pakets erzwingen](#5-neubau-eines-einzelnen-pakets-erzwingen)
- [6. Vollständigen Imagebau erzwingen](#6-vollständigen-imagebau-erzwingen)
- [7. Mehr Informationen](#7-mehr-informationen)

## 1. Vorbereitung

Hier angegebene Pfade basieren auf Vorgaben, die vom Init-Script erzeugt werden. Einige Einträge werden als ```<Platzhalter>``` dargestellt, die lokal angepasst werden müssen. [Siehe Schema](#14-init-skript-ausführen)

**HINWEIS:** Die Wartung der Tuxbox-Builder-VM wird nicht mehr weitergeführt. Falls die VM-Lösung trotzdem verwendet werden soll, springe bitte zu [Schritt 2](#2-image-bauen). Die Tuxbox-Builder-VM enthält normalerweise bereits erforderliche Pakete.

Details und Download: [Tuxbox-Builder](https://sourceforge.net/projects/n4k/files/Tuxbox-Builder)

### 1.1 Erforderliche Host-Pakete installieren

**Hinweis:** Bei Verwendung anderer Distributionen siehe: [Yocto Project Quick Build](https://docs.yoctoproject.org/3.2.4/ref-manual/ref-system-requirements.html#supported-linux-distributions)

Debian 11

```bash
sudo apt-get install -y gawk wget git-core diffstat unzip texinfo gcc-multilib build-essential \
chrpath socat cpio python python3 python3-pip python3-pexpect xz-utils debianutils \
iputils-ping python3-git python3-jinja2 libegl1-mesa pylint3 xterm subversion locales-all \
libxml2-utils ninja-build default-jre clisp libcapstone4 libsdl2-dev doxygen
```

**HINWEIS:** Bei Debian 10 (buster) libcapstone3 verwenden.

#### 1.1.1 Empfohlene Zusatzpakete zur grafischen Unterstützung und Analyse

```bash
sudo apt-get install -y gitk git-gui meld cppcheck clazy kdevelop
```

### 1.2 Git vorbereiten (falls erforderlich)

Das init-Script verwendet Git zum Klonen der Meta-Layer Repositorys. Wenn noch kein konfiguriertes Git vorhanden ist, lege bitte Deine globalen Git-Benutzerdaten an, andernfalls wird während das Script durchläuft, unnötig darauf hingewiesen. 

```bash
git config --global user.email "you@example.com"
git config --global user.name "Dein Name"
```

### 1.3 Init-Skript klonen

```bash
git clone https://github.com/tuxbox-neutrino/buildenv.git && cd buildenv
```

### 1.4 Init-Skript ausführen

```bash
./init && cd poky-3.2.4
```

### 1.5 Struktur der Buildumgebung

Nach [Schritt 1.4](#14-init-skript-ausführen) sollte etwa diese Struktur angelegt worden sein:

```
.buildenv
 ├── dist                          <-- Freigabeordner
 │   └── {DISTRO_VERSION}          <-- hier liegen die erzeugten Images und Pakete (Symlinks zu den Deploy-Verzeichnissen innerhalb der Arbeitsverzeichnisse)
 :
 ├── init                          <-- init-Script
 ├── local.conf.common.inc         <-- globale Benutzerkonfiguration, ist in die benutzerdefinierte Konfiguration inkluiert
 :
 ├── log                           <-- Ordner für Logs
 :
 └── poky-{DISTRO_VERSION}         <-- Nach Punkt 1.4 befindest Du dich hier.
     │
	 :
     ├── <Buildsystem-Kern und Meta-Layer>
     :
     └── build                     <-- Hier liegen die Buildverzeichnisse, nach Schritt 2.2 bist Du in einem der Build-Unterverzeichnisse
         ├── <machine 1>           <-- Buildverzeichnis für Maschinentyp
         │   ├── conf              <-- Ordner für Layer und benutzerdefinierte Konfiguration
         │   │   └── bblayers.conf <-- Konfigurationsdatei für eingebundene Meta-Layer
         │   │   └── local.conf    <-- benutzerdefinierte Konfiguration für einen Maschinentyp
		 │   :
         │   ├── (tmp)             <-- Arbeitsverzeichnis, wird erst beim Bauen angelegt
         │   └── (workspace)       <-- Workspace, wird beim Ausführen von devtool angelegt
         :
         └── <machine x>
```
 
## 2. Image bauen

Stelle sicher, dass Du dich wie im [Schema](#14) gezeigt hier befindest:

```
poky-{DISTRO_VERSION}
```

### 2.1 Box wählen

Liste verfügbarer Geräte anzeigen lassen:

```bash
ls  build
```

### 2.2 Starte Umgebungsskript

Führe das Umgebungsskript für die aus der Liste gewünschte Box einmalig aus! Du gelangst dann automatisch in das passende Buildverzeichnis.

```bash
. ./oe-init-build-env build/<machine>
```

Solange man sich ab jetzt mit der erzeugten Umgebung innerhalb der geöffneten Shell im gewünschten Buildverzeichnis befindet, muss man dieses Script nicht noch einmal ausführen und kannst [Schritt 2.3 ](#23-image-erstellen) Images oder beliebige Pakete bauen.

**Hinweis:** Du kannst auch weitere Shells und damit Buildumgebungen für weitere Boxtypen parallel dazu anlegen und je nach Bedarf auf das entsprechende Terminal wechseln und auch parallel bauen lassen, sofern es dein System hergibt.

### 2.3 Image erstellen

```bash
bitbake neutrino-image
```

Das kann eine Weile dauern. Einige Warnmeldungen können ignoriert werden. Fehlermeldungen, welche die Setscene-Tasks betreffen, sind kein Problem, aber Fehler während der Build- und Package-Tasks brechen den Prozess in den meisten Fällen ab.  [Bitte melde in diesem Fall den Fehler oder teile Deine Lösung](https://forum.tuxbox-neutrino.org/forum/viewforum.php?f=77). Hilfe ist sehr willkommen.

Wenn alles erledigt ist, sollte eine ähnliche Meldung wie diese erscheinen:

```bash
"NOTE: Tasks Summary: Attempted 4568 tasks of which 4198 didn't need to be rerun and all succeeded."
```

<span style="color: green;">Das war's ...</span>


Ergebnisse findest Du unter:

```bash
buildenv/poky-3.2.4/build/<machine>/tmp/deploy
```

oder im Freigabe-Verzeichnis:

```bash
buildenv/dist/<Image-Version>/<machine>/
```

Falls ein Webserver eingerichtet ist, der auf das Freigabe-Verzeichnis zeigt:

```bash
http://localhost/3.2.4
```

## 3. Aktualisierung

Manuelle Aktualisierungen der Pakete sind nicht erforderlich. Dies wird automatisch bei jedem aufgerufenen Target mit Bitbake durchgeführt. Das gilt auch für mögliche Abhängigkeiten. Wenn man die volle Kontrolle über bestimmte Paket-Quellen haben möchte, kann man sich diese für jedes Paket im dafür vorgesehenen Workspace ablegen, siehe [4.4 Pakete](#44-pakete).
Sollten keine Aktualisierungen notwendig sein, werden die Builds automatisch übersprungen.

### 3.1 Image aktualisieren

```bash
bitbake neutrino-image
```
	
### 3.2 Paket aktualisieren

```bash
bitbake <package>
```

### 3.3 Meta-Layer-Repositorys aktualisieren

Die Ausführung des Init-Skripts mit dem ```--update``` Parameter aktualisiert die enthaltenen Meta-Layer auf den Stand der Remote-Repositorys.

```bash
./init --update
```

Falls Du an den Meta-Layern Änderungen vorgenommen hast, sollten angestoßene Update-Routinen des Init-scripts nicht festgeschriebene Änderungen vorübergehend stashen bzw. auf das lokale Repository rebasen. Natürlich kann man seine lokalen Meta-Layer für Meta-Neutrino- und Maschinen-Layer-Repositorys manuell aktualisieren. Konflikte muss man jedoch immer manuell auflösen. 

**Hinweis:** Konfigurationsdateien bleiben im Wesentlichen unberührt, aber mögliche Variablennamen werden migriert. Neue oder geänderte Einstellungen werden nicht geändert. Bitte überprüfe evtl. die Konfiguration.

## 4. Eigene Anpassungen

### 4.1 Konfiguration

Es wird empfohlen, zum ersten Mal ohne geänderte Konfigurationsdateien zu bauen, um einen Eindruck davon zu bekommen, wie der Build-Prozess funktioniert und um die Ergebnisse möglichst schnell zu sehen.
Die Einstellmöglichkeiten sind sehr umfangreich und für Einsteiger nicht wirklich überschaubar. OpenEmbedded insbesondere das Yocto-Project ist jedoch sehr
umfassend dokumentiert und bietet die beste Informationsquelle.

#### 4.1.1 Konfigurationsdateien

> ~/buildenv/poky-3.2.4/build/```<machine>```/conf/local.conf

Diese Datei befindet sich im Buildverzeichnis des jeweiligen Maschinentyps und ist die eigentliche benutzerdefinierte Konfigurationsdatei, welche ursprünglich vom Buildsystem dafür vorgesehen ist. Diese local.conf enthält in dieser Umgebung jedoch nur nur wenige Zeilen und inkludiert eine globale Konfiguration. Diese Datei ist **nur** für den von ihr unterstützten Maschinentyp gültig. Hier kann man deshalb ergänzende Einträge vornhemen, die entsprechend nur für den Maschinentyp vorgesehen sind. [Siehe auch Schema](#14-init-skript-ausführen)

> ~/buildenv/local.conf.common.inc

Diese Datei enthält Einstellungen, die für alle  Maschinentypen zutreffen und wird bei erstmaligen ausführen des Init-Scripts aus der Vorlage ```~/buildenv/local.conf.common.inc.sample``` erzeugt.

Die vom Buildsystem vorgesehene ```./build/<machine>/conf/local.conf``` könnte man zwar so wie es ursprügliche vom Buildsystem vorgesehen ist als primäre Konfigurationsdatei für jeden Maschinentyp separat verwenden, aber das würde den Wartungsaufwand unnötig erhöhen. Deshalb ist ```~/buildenv/local.conf.common.inc``` nur in ```./build/<machine>/conf/local.conf``` inkludiert, 

**Hinweis zu** ```~/buildenv/local.conf.common.inc.sample```**:**  Dies ist nur eine Vorlage und sollte unberührt bleiben, um mögliche Konflikte beim Aktualisieren des Build-Script-Repositorys zu vermeiden und um zu sehen, was sich geändert haben könnte.

Nach einer Aktualisierung des Build-Script-Repositorys könnten neue oder geänderte Optionen hinzugefügt oder entfernt worden sein, die nicht in die inkludierte Konfigurationsdatei übernommen werden. Diesen Fall sollte man in der eigenen Konfiguration berücksichtigen und falls erforderlich prüfen und anpassen.

#### 4.1.2 bblayers.conf

> ~/buildenv/poky-3.2.4/build/```<machine>```/conf/bblayers.conf

Diese Datei wird normalerweise beim erstmaligen ausführen des Init-Skripts angepasst und braucht in der Regel nur angepasst zu werden, wenn man Layer hinzufügen, entfernen oder ersetzen möchte.

#### 4.1.3 Konfiguration zurücksetzen
Wenn Du deine Maschinen-Konfigurationen zurücksetzen möchtest, benenne bitte das conf-Verzeichnis um (Löschen wird nicht empfohlen) und führe das Init-Skript erneut aus.


```bash
~/mv ~/buildenv/poky-3.2.4/build/<machine>/conf ~/buildenv/poky-3.2.4/build/<machine>/conf.01
~/cd ~/buildenv
~/./init
```

### 4.3 Recipes
	
**Bitte ändere nicht die offiziellen Recipes! Dies wird vom Yocto-Team ausdrücklich nicht empfohlen, sofern man nicht direkt an der Entwicklung beiteiligt ist. Man kann aber zum Vervollständigen, Erweitern oder Überschreiben von Meta-Core-Recipes [.bbappend](https://docs.yoctoproject.org/3.2.4/dev-manual/dev-manual-common-tasks.html#using-bbappend-files-in-your-layer)-Dateien verwenden.**

### 4.4 Pakete

Wenn man die volle Kontrolle über einen Paket-Quellcode haben möchte, um z.B. etwas zu fixen oder aktiv zu entwickeln, sollte der Quellcode an dem man arbeiten möchte in den Workspace verschoben werden. Siehe: [Beispiel für Neutrino](#441-neutrino-quellcode-im-workspace-bearbeiten-beispiel)

Siehe [devtool](https://docs.yoctoproject.org/current/ref-manual/devtool-reference.html) und insbesondere [devtool modify](https://docs.yoctoproject.org/current/ref-manual/devtool-reference.html#modifying-an-existing-recipe). Im Workspace hat man die Garantie, dass der Quellcode nicht vom Buildsystem angefasst wird. Beachtet man das nicht, kann es z.B. vorkommen, dass geänderter Quellcode immer wieder gelöscht oder modifiziert wird. Eigene Anpassungen können daher verloren gehen oder inkompatibel werden. In der lokalen Standardkonfiguration ist [rm_work](https://docs.yoctoproject.org/ref-manual/classes.html#ref-classes-rm-work) aktiviert, was dafür sorgt, dass nach jedem abgeschlossenem Bau eines Pakets, das jeweilige Arbeitsverzeichnis aufgeräumt wird, so dass ausser einigen Logs nichts übrig bleiben wird.

#### 4.4.1 Neutrino Quellcode im Workspace bearbeiten (Beispiel)

Diese Vorgehensweise trifft im Wesentlichen auf alle anderen Pakete zu.

```bash
~/buildenv/poky-3.2.4/build/hd61$ devtool modify neutrino-mp
NOTE: Starting bitbake server...54cf81d24c147d888c"
...
workspace            = "3.2.4:13143ea85a1ab7703825c0673128c05845b96cb5"

Initialising tasks: 100% |###################################################################################################################################################################################################| Time: 0:00:01
Sstate summary: Wanted 0 Found 0 Missed 0 Current 10 (0% match, 100% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 83 tasks of which 80 didn't need to be rerun and all succeeded.
INFO: Adding local source files to srctree...
INFO: Source tree extracted to /home/<user>/buildenv/poky-3.2.4/build/hd61/workspace/sources/neutrino-mp
INFO: Recipe neutrino-mp now set up to build from /home/<user>/buildenv/poky-3.2.4/build/hd61/workspace/sources/neutrino-mp
```

Unter ```/buildenv/poky-3.2.4/build/hd61/workspace/sources/neutrino-mp``` befindet sich jetzt der Quellcode für Neutrino. Dort kann man dann daran arbeiten. Das bedeutet, dass das Buildsystem nicht mehr von sich aus vom Remote Git-Repo die Neutrino-Quellen klont, sondern ab jetzt nur noch die lokalen Quellen nutzt, die man selbst verwalten muss. Dabei handelt es sich um ein von devtool angelegtes Git-Repo, in welches man an das Original-Remote-Repository einbinden kann, sofern dies nicht bereits der Fall ist.

Führt man jetzt das aus...

```bash
bitbake neutrino-mp
```

...wird Neutrino ab sofort nur noch vom lokalen Repo im Workspace gebaut werden:


```bash
NOTE: Started PRServer with DBfile: /home/<user>/buildenv/poky-3.2.4/build/hd61/cache/prserv.sqlite3, IP: 127.0.0.1, PORT: 34211, PID: 56838
...
workspace            = "3.2.4:13143ea85a1ab7703825c0673128c05845b96cb5"

Initialising tasks: 100% |###################################################################################################################################################################################################| Time: 0:00:01
Sstate summary: Wanted 122 Found 116 Missed 6 Current 818 (95% match, 99% complete)
NOTE: Executing Tasks
NOTE: neutrino-mp: compiling from external source tree /home/<user>/buildenv/poky-3.2.4/build/hd61/workspace/sources/neutrino-mp
NOTE: Tasks Summary: Attempted 2756 tasks of which 2741 didn't need to be rerun and all succeeded.
```

**Hinweis!** Im speziellen Fall von Neutrino, ist es ratsam nicht nur dessen Quellcode, sondern auch die zugehörige ```libstb-hal``` in den Workspace zu übertragen.

```bash
devtool modify libstb-hal
```

## 5. Neubau eines einzelnen Pakets erzwingen

In einigen Fällen kann es vorkommen, dass ein Target, warum auch immer, abbricht. Man sollte deshalb aber keinesfalls in Panik verfallen und deswegen den Arbeitsordner und den kostbaren sstate-cache löschen. Bereinigungen kann man für jedes Target einzeln vornehmen, ohne ein ansonsten funktionierendes System platt zu machen.

Insbesondere defekte Archiv-URLs können zum Abbruch führen. Diese Fehler werden aber immer angezeigt und man kann die URL überprüfen. Oft liegt es nur an den Servern und funktionieren nach wenigen Minuten sogar wieder.

Um sicherzustellen, ob das betreffende Recipe auch tatsächlich ein Problem hat, macht es Sinn das betreffende Target komplett zu bereinigen und neu zu bauen. Um dies zu erzwingen, müssen alle zugehörigen Paket-, Build- und Cachedaten bereinigt werden.

```bash
bitbake -c cleansstate <target>

```
anschließend neu bauen:

```bash
bitbake <target>
```
	
## 6. Vollständigen Imagebau erzwingen

Das Init-Skript stellt dafür die Option `--reset` zur Verfügung.

```bash
./init --reset
# Anweisungen befolgen
```

Manuell erreichst Du das ebenfalls, indem man das tmp-Verzeichnis im jeweiligem Buildverzeichnis manuell umbenennt. Löschen kann man es nachträglich, wenn man Speicherplatz freigeben will oder sich sicher ist, dass man das Verzeichnis nicht mehr braucht:

```bash
mv tmp tmp.01
```

Anschließend das Image neu bauen lassen:

```bash
bitbake neutrino-image
```

Wenn man den Cache **nicht** gelöscht hat, sollte das Image in relativ kurzer Zeit fertig gebaut sein. Gerade deshalb wird empfohlen, den Cache beizubehalten. Das Verzeichnis wo sich der Cache befindet, wird über die Variable ```${SSTATE_DIR}``` festgelegt und kann in der Konfiguration angepasst werden. 
	
Dieses Verzeichnis ist ziemlich wertvoll und nur in seltenen Fällen ist es notwendig, dieses Verzeichnis zu löschen. Bitte beachte, dass der Build nach dem Löschen des Cache sehr viel mehr Zeit in Anspruch nimmt. 

**Hinweis:** Man kann Bitbake anweisen, keinen Cache zu verwenden.

```bash
bitbake --no-setscene neutrino-image
```

Damit wird beim Bauen kein Cache angelegt.

oder

```bash
bitbake --skip-setscene neutrino-image
```

Damit wird ein bereits vorhandener Cache nicht verwendet.


## 7. Mehr Informationen
Weitere Informationen zum Yocto Buildsystem:

* https://docs.yoctoproject.org/3.2.4/index.html
