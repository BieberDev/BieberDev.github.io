# Smarthome Steuerung mit dem Raspberry
> ## Zigbee Geräte mit dem Raspberry und einem Zigbee Adapter ohne Gateways lokal steuern.

---


## Inhalt
 - Einleitung
 - Docker Installation
 - Verzeichnisstruktur erstellen
    - Konfigurations-Datei für den MQTT-Broker
 - Docker-Compose
 - Node-RED Konfiguration
    - Installation und Einrichtung Zigbee2Mqtt-Nodes

---

## Einleitung
> Für die Steuerung von Zigbee-Geräten wird ein [Zigbee Adapter](https://www.zigbee2mqtt.io/guide/adapters/) benötigt, welcher über den Raspberry mit den Zigbee-Geräten kommunizieren kann.

> Das Tool [Zigbee2MQTT](https://www.zigbee2mqtt.io/) bringt das ganze über [MQTT](https://mosquitto.org/) in ein einheitliches Format.

> Das Empfangen und Senden läuft demnach alles über einen MQTT-Broker, welcher ebenfalls lokal auf dem Raspberry läuft.

> Die Logik, was wann wohin gesendet wird, kann mit dem Tool [Node-RED](https://nodered.org/) konfiguriert werden. Dies bietet eine Web-Oberfläche, in welcher durch Drag&Drop Logiken abgebildet werden können.

--- 

## Docker Installation

> Folgende Befehle über SSH auf dem Raspberry ausführen
```bash
curl -sSL https://get.docker.com | sh
```

Berechtigung anpassen
```bash
sudo usermod -a -G docker $USER
```

docker-compose installieren
```bash
sudo apt-get install libffi-dev libssl-dev
sudo apt install python3-dev
sudo apt-get install -y python3 python3-pip
sudo pip3 install docker-compose
```

Raspberry Neustarten
```bash
sudo reboot
```

---

## Verzeichnisstruktur erstellen
> Um die Einstellungen der Container zu speichern, werden lokal Verzeichnisse erstellt, welche an die Container freigegeben werden.

Unter /home/pi/ wird ein Verzeichnis "DOCKER_DATA" mit folgender Struktur  erstellt:
```
├── MQTT
│   ├── DATA
│   ├── LOG
│   ├── mosquitto.conf (Konfigurations-Datei für den MQTT-Broker)
├── ZIGBEE
├── NODERED
```

---

## Konfigurations-Datei für den MQTT-Broker
Folgender Inhalt muss in die mosquitto.conf-Datei geschrieben werden:
(/home/pi/DOCKER_DATA/MQTT/mosquitto.conf)

```yaml
port 1883
persistence true 
persistence_location /mosquitto/data/

log_type error 
log_type warning
log_type notice
log_type information

log_dest file /mosquitto/log/mosquitto.log
log_timestamp_format %Y-%m-%dT%H:%M:%S

allow_anonymous true
```

---

## Docker-Compose
> Mit dem folgenden docker-compose Skript werden alle benötigten Container erstellt und hochgefahren (MQTT-Broker, Zigbee2Mqtt, NodeRED).

[smarthome-compose.yml](https://bieberdev.github.io/smarthome-compose.yaml)

```yaml
#####################################################################
# All configuration files and data stored in /home/pi/DOCKER_DATA/* #
#                                                                   #
# Start command:                                                    #
# docker-compose -f smarthome-compose.yaml up                       #
#####################################################################
version: '3'
services:
#   Setup:  MQTT - Broker [https://hub.docker.com/_/eclipse-mosquitto]
    mqtt:
        container_name: mqtt
        image: eclipse-mosquitto:latest
        ports:
            - 1883:1883
            - 9001:9001
        volumes:
            - /home/pi/DOCKER_DATA/MQTT/mosquitto.conf:/mosquitto/config/mosquitto.conf
            - /home/pi/DOCKER_DATA/MQTT/DATA:/mosquitto/data
            - /home/pi/DOCKER_DATA/MQTT/LOG:/mosquitto/log
        restart: always

#   Setup:   ZIGBEE2MQTT [https://hub.docker.com/r/koenkk/zigbee2mqtt]
    zigbee:
        container_name: zigbee
        image: koenkk/zigbee2mqtt:latest
        ports:
            - 8080:8080
        volumes:
            - /home/pi/DOCKER_DATA/ZIGBEE:/app/data
            - /run/udev:/run/udev:ro
        devices:
            - /dev/ttyACM0:/dev/ttyACM0
        restart: always
        privileged: true
        environment:
            - TZ=Europe/Berlin

#   Setup:  NODE-RED [https://hub.docker.com/r/nodered/node-red]
    nodered:
        container_name: nodered
        image: nodered/node-red:latest
        ports:
            - 1880:1880
        volumes:
            - /home/pi/DOCKER_DATA/NODERED:/data
        restart: always
```

Das Skript kann als "smarthome-compose.yml" gespeichert und wie folgt aufgerufen werden:
```bash
docker-compose -f smarthome-compose.yml up -d
```

```
-f   compose-Datei
-d   Wird im Hintergrund ausgeführt
```

> Nachdem docker-compose die Images heruntergeladen und die Container gestartet hat, sollten 3 Container aktiv laufen:
```bash
docker ps
```

---

## Node-RED Konfiguration
> Nachdem die Container gestartet sind, sollte unter der IP-Adresse des Raspberrys und dem Port 1883 die Node-RED Web-Oberfläche erscheinen.

Über den [Palette Manager](https://nodered.org/docs/user-guide/editor/palette/manager) können zusätliche Nodes installiert werden.

## Installation und Einrichtung Zigbee2Mqtt-Nodes
Damit Node-RED mit Zigbee2MQTT kommunizieren kann, muss folgende Node installiert werden:

[node-red-contrib-zigbee2mqtt](https://flows.nodered.org/node/node-red-contrib-zigbee2mqtt)

Nach der Installation stehen weitere Nodes zur Verfügung.
In der Zigbee2Mqtt-Bridge Node muss der MQTT-Sever hinterlegt werden.
Dieser ist, wie in dem Compose-File hinterlegt unter ```mqtt://mqtt``` erreichbar. (mqtt = Name des MQTT-Broker Services)

Nach der Konfiguration der Bridge-Node kann man die Zigbee Geräte anlernen, ansteuern und den Status abfragen.

Für das Anlernen und Verwalten der Zigbee-Geräte kann auch das [Frontend](https://www.zigbee2mqtt.io/guide/configuration/frontend.html#nginx-proxy-configuration) von Zigbee2Mqtt verwendet werden.
