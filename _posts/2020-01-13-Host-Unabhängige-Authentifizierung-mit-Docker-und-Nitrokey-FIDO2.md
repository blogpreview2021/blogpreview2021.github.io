---
author: "David Lassig"
layout: "splash"
toc: true
toc_label: "Inhalt"
toc_icon: "file-alt"
header:
  overlay_image: /assets/images/fido_ssh_docker_concept.png
  overlay_filter: 0.5 # same as adding an opacity of 0.5 to a black background
---


# Warum FIDO2 USB-Sticks?

Effektive und effiziente Authentifizierung ist immer Fluch und Segen zugleich. Nutzerkomfort und Sicherheit stehen dabei gefühlt in einem ständigen Konflikt. Sind Passwörter oder sonstige zusätzliche Faktoren zu komplex, sind die Nutzer frustriert und der Helpdesk schnell überlastet. Sind Passwörter und Zusatzfaktoren zu simpel, formt sich ein potenzieller Angriffsvektor.
<br>
Im Unternehmensumfeld haben sich bereits verschiedene Mechanismen wie Smartcards, Single-Sign-On oder isolierte Intranet-Authentifizierung etabliert. Individuelle Nutzer und Privatanwender haben bisher ausser Passwort-Managern noch keine etablierte Lösung zur Hand für elegantes Authentifizierungs-Management.
<br>
---
<br>
Das ist unter anderem auch der Grund warum 2012 die FIDO Alliance (__Fast IDentity Online__) durch PayPal, Lenovo und weiteren zur Etablierung von passwortloser Authentifizierung gegründet wurde.
Mit dem Beitritt von Google, Yubikey und NXP im Jahr 2013 wurden die ersten Implementierungsvorschläge für offene Second-Faktor-Authentifizierungs Implementierungen vorgeschlagen.
<br>
Das führte zu dem momentan aktuellsten Standard der Alliance __FIDO2/FIDO2 U2F__. Dieser wird seit Version 8.2 von OpenSSH zur Authentifizierung unterstützt und führt zu diesem Text. Daneben ist die FIDO-Authentifizierung in erster Linie für browser-basierte Webanwendungen gedacht, welche bisher z.B. von Microsoft und Amazon als zweiter Authentisierungs-Faktor unterstützt wird. Auf [https://webauthn.io/](https://webauthn.io/) kann man seinen FIDO2/FIDO2 U2F Token ausprobieren.
<br>
---
<br>
Neben dem populärsten Vertreter solcher USB-Tokens, den Yubikeys von Yubico, gibt es glücklicherweise auch einen deutschen Anbieter solcher FIDO2/FIDO2 UAF-konformer Sticks. Das ist Nitrokey aus Berlin mit dem Nitrokey FIDO2-Stick. Das Besondere an den Produkten von Nitrokey ist, dass sowohl Software als auch Hardware zu 100% Open Source sind. 


# Warum in Docker Containern?

Es gibt mit der zunehmenden Verbreitung von cloudnativen Netzwerken einen wachsenden Bedarf an containerbasierten Applikationen. Dabei spielen viele Vorteile eine Rolle wie Skalierbarkeit und mächtige Möglichkeiten für Konfigurations-Management. Für dieses Minimalbeispiel nutzen wir folgende Vorteile aus:

  * __host-unababhängige Applikationen__: Nicht jeder hat Lust sich ein OpenSSH >8.4 selbst zu kompilieren und nicht jeder hat bereits ein Ubuntu >20.04 im Betrieb, wo OpenSSH in der notwendigen Version bereits in den Repos ist. So können wir ohne Veränderungen an unserem Betriebssystem bzw. zusätzliche Installationen OpenSSH zur Ausführung bringen.
  * __versionierbare Infrastruktur__: Das Buzzword __Infrastructure-As-Code__ ist in aller Munde. Zu Recht. Dadurch ist es möglich sowohl allein als auch im Team Netzwerk-Aufbau und -Management mit jedem Fortschritt und jeder Veränderung nachvollziehbar archivieren zu können.

Alle diese einzelnen Möglichkeiten und Vorteile könnten jeweils ganze Seiten und Bücher füllen, hier wollen wir uns aber auf die Nutzung von FIDO2 USB-Sticks in einem möglichst einfachen Setup darstellen:

# Docker-basierte OpenSSH FIDO2 Authentifizierung mit Nitrokey FIDO2 Sticks 

Zur Erprobung dieser Funktionalität ohne Änderungen an unserem Host-System vorzunehmen, bzw. diese Funktionalität in Folge elegant auf einen entfernten Server bringen zu können, nutzen wir __docker-compose__ um den __SSH-Client__ und den __SSH-Server__ zu beschreiben und zu testen. Dabei nutzen sowohl Client als auch Server OpenSSH-Version 8.4 zur Ermöglichung der FIDO-Authentifizierung.

> Dazu existiert dieses [Repository](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2) wo die beschriebenen Schritte nachvollzogen werden können.  
   * Dieses kann gecloned werden mit `git clone https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2.git`.
   * Auf dem Host installiert sein müssen `docker > 18.06.0` und `docker-compose > 1.22.0`

![](/assets/images/fido_ssh_docker_concept.png)  

## Docker Container für Client und Server

Ich werde beispielhaft für den Client erklären, was die einzelnen Dateien bedeuten, die notwendig sind um einen docker-compose Container erfolgreich zu starten:

### SSH Client

#### docker-compose.yml

Zentrale Konfigurationsdatei für docker-compose ist die `docker-compose.yml`: ()[https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/blob/main/sshclient/docker-compose.yml]:
  * `build`:  Definition des zu nutzenden Docker Containers bzw. die Umgebung/Ordner mit allen notwendigen Dateien um den Docker Container zu bauen.
  * `volumes`: 
    * `generated_keys` ist sowohl im Container als auch im Host im Dateisystem erreichbar, sodass die erstellten Keys in diesem Ordner persistent auch nach Beendigung des Containers bleiben
    * `/dev/bus/usb` und die weiteren Volumes sind keine Ordner sondern Dateideskriptoren, die dem Container ermöglichen auf USB-Devices die am Hostsystem anstecken, zuzugreifen
  * `privileged`: mit `true` stattet den Container mit maximalen Systemrechten aus
    * aus Sicherheitssicht nicht gut, da die Isolierung der Docker Engine im Host fast aufgehoben wird
    * habe aber keine andere einfache Möglichkeit gefunden dem Container Zugriff auf die USB-Schnittstellen des Hosts zu geben
  * `networks`: beschreibt ein durch mich definiertes Netzwerk/Subnetz im Docker Network, wo dieser Container gestartet wird


```yaml
version: '3.7'

services:
  client:
    restart: "no"
    build:
      context: ssh_build
    hostname: "client.ssh"
    volumes:
      - ./generated_keys/:/root/generated_keys/
      - /dev/bus/usb:/dev/bus/usb
      - /sys/bus/usb/:/sys/bus/usb/
      - /sys/devices/:/sys/devices/ 
      - /dev/hidraw0:/dev/hidraw0
    privileged: true

networks:
  default:
    external:
      name: fido-ssh-test
```

#### Dockerfile

Hier beschreibe ich die Funktion inline als Kommentar in [Dockerfile](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/blob/main/sshclient/ssh_build/Dockerfile):

```bash
# nutze ein Ubuntu 20.04 als Container-Basis
FROM ubuntu:20.04
# alle Pakete updaten und upgraden
RUN apt update && apt -y upgrade
# installiert openssh aus Standard-Repos (ist in Ubuntu 20.04 OpenSSH 8.4)
RUN apt install -y openssh-server

# hinzufuegen eines Skriptes welches zum Startzeitpunkt des Containers ausgefuehrt wird
COPY ./docker_entrypoint.sh /tmp/docker_entrypoint.sh
RUN chmod +x /tmp/docker_entrypoint.sh

# zusaetzlicher non-root user
RUN useradd -ms /bin/bash user

# Starttrigger des Containers mit hinzugefuegtem Skript
ENTRYPOINT ["/tmp/docker_entrypoint.sh"]
```

#### docker_entrypoint.sh

Das Skript zur Beschreibung des Starttriggers [docker_entrypoint.sh](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/blob/main/sshclient/ssh_build/docker_entrypoint.sh):

```bash
#!/bin/bash

# Start des SSH-Servers (ist beim Client nicht zwingend erforderlich, ist eher exemplarisch fuer Nutzung von docker_entrypoint Skript)
/etc/init.d/ssh start
# haelt Docker Container am "Laufen", da sonst nach Ausfuehrung aller Befehle der Container sich beenden wuerde
tail -f /dev/null
```

### SSH Server

Der Server folgt ähnlichem Muster, kopiert aber zusätzlich noch eine `sshd_config` in den Container, die den Betrieb des SSH Servers konfiguriert und die Nutzung von zertifikats-basierter Authentifizierung zulässt.

## Erstellung Docker-Netzwerk und Container-Start

Wir haben gerade gesehen, dass wir für unsere Container ein eigenes Netzwerk `fido-ssh-test` nutzen wollen. Dieses müssen wir jetzt mit Docker erstellen:

```bash
sudo docker network create fido-ssh-test
```

Nun können wir auch die docker-compose Beschreibungen bauen und zu lauffähigen Instanzen machen. Als erstes der SSH-Server:

```bash
cd sshserver
sudo docker-compose build
```

![](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/ssh_server_build.png)

Den erfolgreichen Build können wir jetzt starten:

```bash
sudo docker-compose up
```

![](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/ssh_server_up.png)


Jetzt verfahren wir beim Client genauso aber machen den Build und das Starten in einem Schritt und führen den Container im Hintergrund aus:

```bash
cd ..
cd sshclient
sudo docker-compose up -d
```

Server und Client laufen und lassen sich im Docker-Netzwerk nachweisen:


```bash
sudo docker network inspect fido-ssh-test
```

![](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/docker_network_online.png)

## Erstellung des SSH Keys

Jetzt können wir in dem SSH-Client den privaten und den öffentlichen Key bauen zum Aufbau einer zertifikat-basierten Authentifizierung.

```bash
sudo docker exec -it sshclient_client_1 /bin/bash
```

Nun sind wir im Container und können mit `ssh-keygen` die Zertifikate gegen den Nitrokey FIDO2-Stick generieren:

```bash
incontainer $ ssh-keygen -t ecdsa-sk -O resident -f generated_keys/first_fido_ssh_key
```

![](/assets/images/nitrokey.gif) 

![]https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/ssh_client_key_creation.png()

  * Mit der Option `-t` wird der Typ des kryptografischen Verfahrens für das Schlüsselpaar definiert. Für FIDO2-basierte Authentifizierung mit Nitrokey ist nach meinem Kenntnisstand bisher `ecdsa-sk` unterstützt.


  * Mit der Option `-O` können wir unter anderem FIDO-Stick spezifische Parameter angeben, z.B.:
    * `-O resident`: ermöglicht für FIDO2-Sticks das Speichern des Private Key auf dem Stick selbst (wird gleich erklärt)
    * `-O no-touch-required`: ermöglicht Authentifizierung mit dem Stick ohne Berührung
    * `-O device`: manuelle Spezifikation des FIDO Devices 
  

Der Public-Teil des erstellten Schlüsselpaares `first_fido_ssh_key.pub` muss jetzt in den Ordner `sshserver/authorized_keys` verschoben werden:

```bash
mv generated_keys/first_fido_ssh_key.pub ../sshserver/authorized_keys
```

Nach dem Hinzufügen des Schlüssels in das Verzeichnis des Servers, muss dieser Container einmal neu gestartet werden:

```bash
cd ..
cd sshserver
sudo docker-compose restart
```

## SSH-Verbindung mit dateibasiertem Private Key

Neben dem nachfolgenden Verwahren des Keys auf dem FIDO2-Token selbst, können wir uns in gewohnter Weise mit dem dateibasierten Private Key mit dem Nitrokey zum Server verbinden:

```bash
sudo docker exec -it sshclient_client_1 /bin/bash
incontainer $ cd
incontainer $ ssh -i generated_keys/first_fido_ssh_key user@sshserver_server_1
```

![](/assets/images/nitrokey.gif) 

![](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/ssh_connect.png)


## SSH-Verbindung mit token-basiertem Private Key

Mit dem Parameter `-O resident` haben wir bereits ein Schlüsselpaar erstellt welches geeignet ist direkt auf dem Nitrokey FIDO2-Stick gespeichert zu werden. Dazu wird das SSH-Tool `ssh-agent` genutzt, welches für die einfache Organisation von mehreren SSH-Keys gedacht ist. Weiterhin ist dafür das Setzen eines PINs auf dem Nitrokey FIDO2-Stick erforderlich.



### Setzen des PINs mit Chrome Browser

Der Google Chrome Browser bringt "von Haus aus" Funktionalitäten zur Kommunikation mit FIDO2-Tokens mit. Die Funktionalität kann im Chrome Browser gemäß diesem [Nitrokey-Beitrag](https://support.nitrokey.com/t/nitrokey-fido2-pin-creating/2062) gefunden werden oder kann von dem Screenshot abgeleitet werden:

![](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/nitrokey_set_pin.png)



### Speichern des Private Key auf dem Nitrokey FIDO2-Token

Um unseren erstellten `first_fido_ssh_key` nun auf dem Token abzulegen müssen wir `ssh-agent` starten und mit `ssh-add` den Schlüssel hinzufügen.

```bash
sudo docker exec -it sshclient_client_1 /bin/bash
incontainer $ eval `ssh-agent -s`
incontainer $ ssh-add -K generated_keys/first_fido_ssh_key
```

![](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/ssh_keyadd.png)

Nachdem der PIN eingegeben wurde, wird der Private Key auf dem Stick hinterlegt. In Chrome können wir dies jetzt überprüfen:

![](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/chrome_key.png)

Weiterhin können über `ssh-add -K -L` die momentan verfügbaren Keys betrachtet werden.

### Verbinden zum Server

Um nachzuweisen, dass der Private Key aus dem Nitrokey kommt und nicht dateibasiert ist, löschen wir die vorhandenen Keys, setzen den Container zurück und verbinden uns mit dem gespeichertem Private Key aus dem Nitrokey:

```bash
sudo docker-compose down
sudo docker-compose up -d
sudo docker exec -it sshclient_client_1 /bin/bash
incontainer $ cd
incontainer $ eval `ssh-agent -s`
incontainer $ ssh-add -K
incontainer $ ssh user@sshserver_server_1
```

![](/assets/images/nitrokey.gif) 

![](https://github.com/cyberinnovationhub/openssh-docker-nitrokey-fido2/raw/main/screenshots/ssh_connect_fresh.png)

# Schlussfolgerungen

Wir können sehen, dass ein reiner FIDO2-Token, in diesem Falle von Nitrokey eine gute Möglichkeit darstellen kann, das Sicherheitsniveau zum Zugriff und Nutzung der eigenen Infrastruktur und von bereitgestellten Webservices, bei relativ geringem Aufwand, zu erhöhen. Insbesondere finanziell ergibt sich hier gegenüber Tokens mit verschlüsseltem Storage und verschlüsseltem Passwortspeicher ein klarer Vorteil.

Natürlich skaliert dieses händische Vorgehen nicht auf die Konfiguration und die Pflege einer großen Anzahl solcher Tokens. Dafür werden wir in naher Zukunft vorhandene Lösungen erproben, die vielleicht auch zu einem neuen Blogpost reifen. :)

Weiterhin zeigt sich die Erkenntnis, dass Docker auch sehr gut auf einem kleinen Host-System (Laptop, Desktop) genutzt werden kann um diverse experimentelle Setups (auch mit Abhängikeiten an USB-Schnittstellen) zu erproben und diese dann mit relativ geringem Aufwand auf größere Infrastruktur zu übertragen. Auch zu diesen Themen, Orchestration und skalierbare Serverkonfiguration, werden weitere Blogposts folgen.

Bis dahin, Happy Hacking :)

![](/assets/images/happyhacking.gif)


Achso, und ich/wir haben die Weisheit natürlich nicht mit Löffeln gefressen. Über konstruktive Kritik, dass man beim Vorgehen bestimmte Sachen besser oder anders machen kann, würden wir uns total freuen. #SharingIsCaring


# Quellen

  * [https://chemnitzer.linux-tage.de/2019/media/programm/folien/223.pdf](https://chemnitzer.linux-tage.de/2019/media/programm/folien/223.pdf)  
  * [https://chemnitzer.linux-tage.de/2019/media/programm/folien/223.pdf](https://chemnitzer.linux-tage.de/2019/media/programm/folien/223.pdf)
  * [https://forum.yubico.com/viewtopic61c9.html?p=8058](https://forum.yubico.com/viewtopic61c9.html?p=8058)
  * [https://fidoalliance.org/](https://fidoalliance.org/)
