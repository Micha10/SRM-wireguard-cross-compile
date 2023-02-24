# SRM-wireguard-cross-compile
Wireguard für Synology Router RT6600ax cross kompilieren

Vielleicht mag die Beschreibung umständlich erscheinen, aber ich habe das auch das erste Mal gemacht. Verbesserungen sind willkommen.

Am besten eine Linux-VM verwenden oder notfalls ein installiertes Linux

```
sudo su
apt-get update
apt-get upgrade
apt-get install git
mkdir /toolkit
cd /toolkit
```

Download der Synology Toolchain für die Cypress CPU

[ds.cypress-1.3.dev.tgz](https://global.synologydownload.com/download/ToolChain/toolkit/1.3/cypress/ds.cypress-1.3.dev.tgz)

[ds.cypress-1.3.env.tgz](https://global.synologydownload.com/download/ToolChain/toolkit/1.3/cypress/ds.cypress-1.3.env.tgz)

[synogpl-9346-cypress.tbz](https://global.synologydownload.com/download/ToolChain/Synology%20NAS%20GPL%20Source/1.3-9346/cypress/synogpl-9346-cypress.tbz)

Der Developerguide ist für die NAS-Systeme geschrieben, ist aber trotzdem informativ

[DSM_Developer_Guide_7_enu.pdf](https://global.download.synology.com/download/Document/Software/DeveloperGuide/Os/DSM/All/enu/DSM_Developer_Guide_7_enu.pdf)

Entweder entpackt man die Archive manuell oder per EnvDeploy. Das Script braucht dann auch das Archiv ```base_env-7.2.txz```, obwohl das für das Kompilieren nicht benötigt wird.
Möchte man also die pkgscripts verwenden, bitte auch das Archiv [base_env-7.2.txz](https://global.synologydownload.com/download/ToolChain/toolkit/7.2/base/base_env-7.2.txz) herunterladen.

```
mkdir toolkit_tarballs
cd toolkit_tarballs
mkdir /toolkit/build_env
mkdir /toolkit/build_env/ds.cypress-1.3
```

Die heruntergeladenen Archive auf das Linuxsystem nach ```/toolkit/toolkit_tarballs``` kopieren.

Entweder extrahiert man die Dateien manuell nach ```/toolkit/build_env``` oder man clont dieses Repo und verwendet die im DSM_DeveloperGuide_7 verwendeten Scripts /EnvDeploy. Die originalen Skripte könnten nicht verwendet werden, da sie die Cypress-Architektur nicht unterstützen. 

-------------
## Nötige Änderungen an den pkgscripts:
(in diesem Repo sind folgende Änderungen des Originalrepos enthalten)

Klonen von
https://github.com/SynologyOpenSource/pkgscripts-ng.git

Wechseln auf den Branch 7.2  (!)
 
 ```git checkout DSM7.2```

include/platforms:

```AllPlatformOptionNames``` hinzufügen von ```cypress```


 Hinzufügen der Zeile

 ```
 cypress  	IPQ6018			linux-3.10.x-bsp	IPQ6018
```


include\toolkit.config

```AvailablePlatform_7_2=...```
Hinzufügen von ```cypress```


\include\python\toolkit.py

Da die DSM-Archive eine txz-Extension haben, die von SRM aber tgz, muss folgendes geändert werden:

von
```
    def get_env_tarball_name(self, platform):
        return 'ds.%s-%s.env.txz' % (platform, self.version)

    def get_dev_tarball_name(self, platform):
        return 'ds.%s-%s.dev.txz' % (platform, self.version)

```

nach

```

    def get_env_tarball_name(self, platform):
        return 'ds.%s-%s.env.tgz' % (platform, self.version)

    def get_dev_tarball_name(self, platform):
        return 'ds.%s-%s.dev.tgz' % (platform, self.version)

```

\include\install

```
CheckTarballExist() {
	local proj=$1
	local tarball=${proj}.tgz
...
```

```
CreateTarball() {
...
	if [ ! -z "$haveFile" ]; then
		echo "Create ${proj}.tgz ..."
		( cd "$TmpInstDir"; XZ_OPT=-3 tar cJpf "$TarBallDir/${proj}.tgz" * )
		echo "Done"
...
```

Erstellen einer neuen Datei include/platform.cypress mit dem Inhalt der extrahierten /env64.mak

include\project.depends:

Hinzufügen der Zeile 

```cypress="linux-3.10.x"```

Da wir die Dateien selber heruntergeladen haben, sollte dem Script EnvDeploy immer der Parameter ```-D``` angehängt werden, damit NICHT automatisch immer wieder heruntergeladen wird. Da die Archive über ein GB groß sind, spart man sich so eine Menge an Zeit.

Da die Skripte für DSM 7.2 geschrieben wurden, erwarten sie auch die Version im Dateinamen. Daher müssen alle Dateien umbenannt werden, um zum Schema zu passen

```
cd /tarball/toolkit_tarballs
mv ds.cypress-1.3.dev.tgz ds.cypress-7.2.dev.tgz
mv ds.cypress-1.3.env.tgz ds.cypress-7.2.env.tgz
```

Nun kann man das Script aufrufen.

```
./EnvDeploy -v 7.2 -p cypress -D
```

Es dauert eine Weile, bis alles entpackt ist.

Nun wechselt man mit chroot in das Verzeichnis

```
cd /toolkit/build_env
chroot ds.cypress-7.2
```

==============

# (Optional) Test des Crosscompilers

In einem neuen Terminal (ohne chroot) im Verzeichnis ```/toolkit/build_env/ds.cypress-7.2/tmp``` folgende Dateien erstellen:


Name "Makefile"
```
include	/env.mak
EXEC=	hello
OBJS=	hello.o
all:	$(EXEC)
$(EXEC):	$(OBJS)
				$(CC)	$(CFLAGS)	$<	-o	$@	$(LDFLAGS)
install:	$(EXEC)
				mkdir	-p	$(DESTDIR)/usr/bin/
				install	$<	$(DESTDIR)/usr/bin/
clean:
				rm	-rf	*.o	$(EXEC)
```

Name "hello.c"

```
#include <stdio.h>

void main (void) {
    printf("***** H ello  W orld ****");
}

```

Compilieren mit dem Befehl
```make```

Kommt die Fehlermeldung ```make: /usr/local/aarch64-unknown-linux-gnu/bin/aarch64-unknown-linux-gnu-ccache-gcc: No such file or directory``` editiert man die Datei /toolkit/build_env/ env32.mak. 

In der Zeile beginnend mit "CC=" steht ```aarch64-unknown-linux-gnu-ccache-gcc```. Hier löscht man ```-ccache```, so daß die Zeile lautet:

```
CC=/usr/local/aarch64-unknown-linux-gnu/bin/aarch64-unknown-linux-gnu-gcc
```

Nun sollte das Beispielprogramm compilierbar sein.
Versucht man es zu starten, sollte KEINE Ausgabe erfolgen, sondern ein Fehler wie ```bash: ./hello: cannot execute binary file: Exec format error```

Kopiert man aber das hello-File per ssh auf den Router, kann man es ausführen und die Begrüßung erscheint.

------------

# Nun kommen wir zu Wireguard.

```
cd /toolkit/build_env/ds.cypress-7.2/tmp
git clone https://git.zx2c4.com/wireguard-tools
```

Im chroot-Terminal:
```
cd /tmp/wireguard-tools/src
make
```

Wahrscheinlich kommt folgende Fehlermeldung
```
CHROOT@cypress-build[/tmp/wireguard-tools/src]# make
  CC      wg.o
  CC      config.o
In file included from config.c:18:
containers.h:16:10: fatal error: linux/wireguard.h: No such file or directory
   16 | #include <linux/wireguard.h>
      |          ^~~~~~~~~~~~~~~~~~~
compilation terminated.
make: *** [<builtin>: config.o] Error 1
```

Im Makefile die Zeile ```IDIR = ./uapi/linux/```
hinzufügen und nach der Zeile ```CFLAGS ?= -O3``` einfügen ```CFLAGS += -I$(IDIR)```

Make sollte nun durchlaufen

```
CHROOT@cypress-build[/tmp/wireguard-tools/src]# make
  CC      config.o
  CC      curve25519.o
  CC      encoding.o
  CC      genkey.o
  CC      ipc.o
  CC      pubkey.o
  CC      set.o
  CC      setconf.o
  CC      show.o
  CC      showconf.o
  CC      terminal.o
  LD      wg

```

... und endlich hat man das WG-File für Wireguard.

Einfach auf den Router kopieren und Konfigurieren.

Zur Konfiguration siehe [Teil 2](README_CONFIG.md)