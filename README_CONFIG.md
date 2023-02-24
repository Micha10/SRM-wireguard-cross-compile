# Konfiguration Wireguard

Kopieren der selbstcompilierten wd-Datei auf den Router per ssh.

Entweder den Pfad anpassen, so daß die wd-Datei gefunden werden kann, oder den absoluten Pfad angeben oder einen Alias setzen
```
alias wg=/volume1/WireGuard/bin/wg
```

Generieren der Keys für den Router
```
mkdir /etc/wireguard
cd /etc/wireguard
wg genkey | tee privatekey | wg pubkey > publickey
ip link add wg0 type wireguard
```

Keys für einen Peer generieren
```
wg genkey | tee privatekey_client | wg pubkey > publickey_client
```


Die IP ist die VPN-Range die der Router bereitstellen soll.
```
ip addr add 192.168.34.1/24 dev wg0
wg set wg0 private-key /etc/wireguard/privatekey listen-port 47111 
ip link set wg0 up
```
Jetzt wird dem Router die Info gegeben, daß ein bestimmter Client eine VPN-Verbindung verwenden darf.
Aus der oben generierten Datei den publickey des clients einfügen. Der Endpoint ist die aktuelle IP-Adresse des Clients
```
wg set wg0 peer __HIER_DEN_publickey_client__EINFUEGEN allowed-ips 192.168.34.0/16 endpoint 192.168.9.4:47111
wg
```

Damit man auch von außen Zugriff bekommen, muß in der Synology-Firewall ein Port geöffnet werden.
```
Systemsteuerung / Sicherheit / Firewall

Erstellen

Protokoll: UDP
Quelle Netzwerkschnittstelle: alle
Ip-Adresse: alle
Ports: alle

Ziel Netzwerkschnittstelle: alle
Ip-Adresse: SRM
Ports: 47111

zulassen
```

## Smartphone:

Wireguard installieren. 

Hinzufügen (rechts unten + / Neu erstellen) 
```
Name: Wie die Verbindung heißen soll

Privater Schlüssel: Das ist der private Schlüssel aus der oben generierten Datei privatekey_client

Der öffentliche Schlüssel wird automatisch aus dem privaten generiert.

Adressen: z.B. 192.168.34.5, also eine Adresse aus dem Bereich, der beim Router hinterlegt wurden
```

Nun fügt man mit "Gegenüber hinzufügen" die Daten des Servers ein.

```
öffentlicher Schlüssel: Das ist der publicKey des Routers aus der Datei publickey

Endpunkt: Hier die Routeradresse eingeben. Hat man eine dynamische IP - wie die meisten Privatkunden - braucht man hier seine DynDNS-Adresse, gefolgt von der Portnummer hinter einem Doppelpunkt
z.B. meine-dynamische-ip:47111

```

Speichern und ausprobieren.

--------------------

TODO: Daten persistent speichern. Momentan sind nach einem Routerneustart die Infos weg