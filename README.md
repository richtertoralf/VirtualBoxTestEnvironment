# VirtualBoxTestEnvironment
Testumgebungen in VirtualBox bauen
>Ab und zu entwickle ich Software, mit der zwischen verschiedenen Maschinen Daten ausgetauscht werden soll, also Software für verteilte Systeme.
>Für die Softwareentwicklung und das Testen dieser Software benötige ich Testumgebungen, die oft aus mehreren Computern und Routern bestehen.
>Bei längerfristigen Projekten nutze ich für das Ausrollen der Testumgebung Vagrant oder Ansible. Manchmal muss es aber auch mal schnell per Hand gemacht werden. Dazu im Folgenden paar Beispiele

## einfacher virtueller Debian-Router
![simple Testumgebung / Router mit zwei Subnetzen und DHCP](https://github.com/richtertoralf/VirtualBoxTestEnvironment/blob/24660940c16c5e3eb00373b97982ac7ac37586ea/VB_TestEnvironment_01.jpg)

Anstatt der alten Bezeichnungen der Netzwerkschnittstellen (eth0, eth1 und eth2) oben im Bild, verwende ich die moderneren "Predictable Network Interface Names", die anhand ihrer physischen Eigenschaften und Position auf dem Motherboard benannt werden. Dies führt zu konsistenteren und vorhersehbareren Namen, insbesondere in Umgebungen mit mehreren Netzwerkschnittstellen oder bei der Verwendung von Virtualisierungstechnologien.  
eth0 -> enp0s3  
eth1 -> enp0s8  
eth2 -> enp0s9  

### Netzwerkadapter planen und konfigurieren
**Das Folgende als Benutzer `root`erledigen: `su -`**  
In Debian wird die Datei `/etc/network/interfaces` verwendet, um Netzwerkschnittstellen zu konfigurieren.
```
# WAN interface
auto enp0s3
iface enp0s3 inet dhcp
iface enp0s3 inet6 auto

# LAN interface 1
auto enp0s8
iface enp0s8 inet static
    address 192.168.100.1
    netmask 255.255.255.0
iface enp0s8 inet6 static
    address fd00::c0a8:6401/64

# LAN interface 2
auto enp0s9
iface enp0s9 inet static
    address 192.168.200.1
    netmask 255.255.255.0
iface enp0s9 inet6 static
    address fd00::c0a8:c801/64
```
Jetzt, den Netzwerkdienst neu starten, damit diese Änderungen übernommen werden:
```
systemctl restart networking
```
Mit `ifup --all --verbose` kannst du vorher einen Test durchführen.  
**enp0s3**: Diese Schnittstelle ist mit dem Internet verbunden (WAN) und bezieht ihre IPv4-Adresse über DHCP. Die IPv6-Adresse wird automatisch konfiguriert.

**enp0s8 und enp0s9**: Diese internen Schnittstellen sind mit den internen Netzwerken verbunden (LAN). Sie sind sowohl für IPv4 als auch für IPv6 statisch konfiguriert.

Die IPv4-Adressen 192.168.100.1 und 192.168.200.1 fungieren als Gateway-Adressen für die internen Netzwerke. Die entsprechenden IPv6-Adressen fd00::c0a8:6401 und fd00::c0a8:c801 sind Unique Local Addresses (ULAs) für die internen Netzwerke.

Die IPv4-Adressen wurden für die Umrechnung in IPv6-Adressen entsprechend dem Unique Local Address-Bereich (fd00::/8) verwendet, indem ich einfach die IPv4-Adressen in hexadezimale Notation umgewandelt habe. Zum Beispiel wird die IPv4-Adresse 192.168.100.1 in IPv6-Adresse fd00::c0a8:6401 umgewandelt.  
| Dezimal | Hexadezimal |
|---------|-------------|
| 192     | C0          |
| 168     | A8          |
| 100     | 64          |
| 1       | 01          |


### IP forwarding / IP-Weiterleitung aktivieren
Aktivieren der IP-Weiterleitung im Linux Kernel
>Das Aktivieren der IP-Weiterleitung im Linux Kernel ist erforderlich, um Datenverkehr zwischen den internen Netzwerken und dem Internet weiterzuleiten. Ohne IP-Weiterleitung würde der Router den Datenverkehr nicht zwischen den verschiedenen Schnittstellen weiterleiten können, was zu einer isolierten Kommunikation zwischen den Netzwerken führen würde.

Editiere die Datei `/etc/sysctl.conf` 
```
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
net.ipv6.conf.all.forwarding=1
```
Mit `sysctl -p` solltest du jetzt diese Ausgabe bekommen:  
```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

### Konfigurieren der Routing-Software
Folgende drei Varianten kannst du nutzen: `iptables`, `nftables` oder `Firewalld`
#### iptables (traditionell)
```
apt-get install iptables iptables-persistent

# IPv4-Regeln
iptables -A FORWARD -j ACCEPT
iptables -t nat -A POSTROUTING -j MASQUERADE

# IPv6-Regeln
ip6tables -A FORWARD -j ACCEPT
ip6tables -t nat -A POSTROUTING -j MASQUERADE

# Speichern der Regeln
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6
```
#### nftables
Ist mir jetzt gerade zu umständlich ;-)

#### Firewalld
```
apt-get install firewalld
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --add-masquerade --permanent   # IPv4 NAT
firewall-cmd --add-masquerade --permanent --zone=trusted --add-interface=enp0s8   # IPv4 NAT für internes Netzwerk
firewall-cmd --add-masquerade --permanent --zone=trusted --add-interface=enp0s9   # IPv4 NAT für internes Netzwerk
firewall-cmd --add-masquerade --permanent --ipv6   # IPv6 NAT
firewall-cmd --reload
```

## einfacher DHCP-Server
>Ein DHCP-Server ermöglicht die automatische Zuweisung von IP-Adressen und anderen Netzwerkkonfigurationsinformationen an Clients in einem Netzwerk. Dnsmasq ist eine leichtgewichtige und vielseitige Software, die DHCP-Dienste sowie DNS-Forwarding und Caching bereitstellen kann, was sie meines Erachtens ideal für den Einsatz in kleinen Netzwerken oder Testumgebungen macht.

### dnsmasq
```
sudo apt install dnsmasq
```
Konfigurationsdatei `/etc/dnsmasq.conf` editieren  
bzw. einfach das Folgende oben einfügen:

meine Konfiguration enthält zusätzlich noch Festlegungen zur lokalen Domain und vergibt feste IP-Adressen für einen Server im LAN 1:
```
# LAN 1
interface=enp0s8  # 1. Schnittstelle, auf der dnsmasq lauscht
domain-needed
bogus-priv
local=/intnet100/
domain=intnet100
listen-address=192.168.100.1  # IP-Adresse des LAN-Interfaces
listen-address=fd00::c0a8:6401  # IPv6-Adresse des LAN-Interfaces
dhcp-range=enp0s8,192.168.100.100,192.168.100.200,24h  # IPv4 DHCP-Adressbereich und Lease-Zeit
dhcp-range=enp0s8,fd00::c0a8:6464,fd00::c0a8:64c8,24h  # IPv6 DHCP-Adressbereich und Lease-Zeit
enable-ra
# dhcp-option für LAN 1 (IPv4 und IPv6)
dhcp-option=enp0s8,option:dns-server,192.168.100.1
dhcp-option=enp0s8,option:dns-server,[fd00::c0a8:6401]
dhcp-option=enp0s8,option:router,192.168.100.1
dhcp-option=enp0s8,option:router,[fd00::c0a8:6401]
# static ip für einzelne server
dhcp-host=08:00:27:7a:9d:a7,ubuntu2204server01,192.168.100.101
dhcp-host=08:00:27:7a:9d:a7,ubuntu2204server01,[fd00::c0a8:6465]

# LAN 2
interface=enp0s9  # 2. Schnittstelle, auf der dnsmasq lauscht
domain-needed
bogus-priv
local=/intnet200/
domain=intnet200
listen-address=192.168.200.1  # IP-Adresse des LAN-Interfaces
listen-address=fd00::c0a8:c801  # IPv6-Adresse des LAN-Interfaces
dhcp-range=enp0s9,192.168.200.100,192.168.200.200,24h  # IPv4 DHCP-Adressbereich und Lease-Zeit
dhcp-range=enp0s9,fd00::c0a8:c864,fd00::c0a8:c8c8,24h  # IPv6 DHCP-Adressbereich und Lease-Zeit
enable-ra
# dhcp-option für LAN 2 (IPv4 und IPv6)
dhcp-option=enp0s9,option:dns-server,192.168.200.1
dhcp-option=enp0s9,option:dns-server,[fd00::c0a8:c801]
dhcp-option=enp0s9,option:router,192.168.200.1
dhcp-option=enp0s9,option:router,[fd00::c0a8:c801]
```
Speichere die Datei und starte dnsmasq neu, damit die Änderungen wirksam werden:
```
# Syntax der /etc/dnsmasq.conf testen:
dnsmasq --test
# dnsmasq neu starten:
systemctl restart dnsmasq
```
**Hier nochmal eine kurze Zusammenfassung der Konfiguration:**   
**LAN 1:**  
Die IPv4-Adresse für das LAN-Interface ist 192.168.100.1, und der DHCP-Bereich ist von 192.168.100.100 bis 192.168.100.200.  
Anzahl der Adressen = 192.168.100.200 - 192.168.100.100 + 1 = 101 Adressen  
Die IPv6-Adresse für das LAN-Interface ist fd00::c0a8:6401, und der DHCP-Bereich ist im fd00::c0a8:6464 bis fd00::c0a8:64c8 Bereich.  
Der Bereich fd00::c0a8:6464 bis fd00::c0a8:64c8 enthält 193 Adressen.  
Diese Adressen gehören zum Unique Local Address (ULA)-Bereich und sind für private IPv6-Netzwerke bestimmt.  
**LAN 2:**  
Die IPv4-Adresse für das LAN-Interface ist 192.168.200.1, und der DHCP-Bereich ist von 192.168.200.100 bis 192.168.200.200.  
Die IPv6-Adresse für das LAN-Interface ist fd00::c0a8:c801, und der DHCP-Bereich ist im fd00::c0a8:c864 bis fd00::c0a8:c8c8 Bereich.

wichtig ist:  

**enable-ra - Router Advertisement (RA) einschalten**  

Diese Option aktiviert das Senden von Router Advertisements (RA) für IPv6 im Netzwerk. RA-Nachrichten sind wichtig für IPv6-Netzwerke, da sie Clients darüber informieren, dass ein Router verfügbar ist und welche Adresskonfigurationsparameter (wie z. B. die Präfixlänge) verwendet werden sollen. RA-Nachrichten sind für den ordnungsgemäßen Betrieb von IPv6-Netzwerken unerlässlich, insbesondere wenn Stateless Address Autoconfiguration (SLAAC) verwendet wird, um IPv6-Adressen automatisch zuzuweisen.  

Das Folgende ist optional:  

**dhcp-authoritative - authoritative DHCP mode**  

Diese Option gibt dnsmasq die letzte Autorität für DHCP-Anfragen auf dem jeweiligen Interface. Wenn mehr als ein DHCP-Server im Netzwerk vorhanden ist, könnte es zu Konflikten kommen, wenn beide Server auf Anfragen antworten. Durch die Verwendung des dhcp-authoritative-Flags stellt dnsmasq sicher, dass seine Antworten auf DHCP-Anfragen priorisiert werden. Dies ist besonders wichtig, wenn dnsmasq als zentraler DHCP-Server in Ihrem Netzwerk verwendet wird, um sicherzustellen, dass die Konfiguration einheitlich ist und um Konflikte mit anderen DHCP-Servern zu vermeiden.  

**domain-needed**  

Diese Option stellt sicher, dass dnsmasq nur auf Anfragen antwortet, die eine Domain enthalten. Das bedeutet, dass dnsmasq nur auf Anfragen reagiert, die einen vollständig qualifizierten Domainnamen (FQDN) enthalten, und nicht auf einzelne Hostnamen ohne Domain. Dies hilft, das unerwartete Auftreten von DNS-Anfragen für ungültige Domains zu reduzieren und sicherzustellen, dass nur gültige DNS-Anfragen beantwortet werden.  

**bogus-priv**  

Diese Option verhindert, dass dnsmasq Antworten auf Anfragen für private IP-Bereiche liefert. DNS-Anfragen für private IP-Adressen sollten normalerweise nicht von externen DNS-Servern beantwortet werden, da dies potenzielle Sicherheitsrisiken darstellen kann. Die bogus-priv-Option sorgt dafür, dass dnsmasq solche Anfragen blockiert und nur auf Anfragen für öffentliche Domains antwortet.  

**local=/intnet100/**  

Diese Einstellung definiert eine lokale DNS-Domäne für Ihr Netzwerk. Der Parameter /intnet100/ gibt an, dass alle Hosts in Ihrem Netzwerk unterhalb dieser Domainnamenshierarchie organisiert sind. Zum Beispiel wird ein Host mit dem Namen host1 unter der Domain host1.intnet100 erreichbar sein. Dies ist nützlich für die interne Namensauflösung innerhalb Ihres lokalen Netzwerks.  

**domain=intnet100**  

Diese Einstellung legt die Standard-Domain für unvollständige Hostnamen fest. Wenn eine DNS-Anfrage einen unvollständigen Hostnamen enthält (zum Beispiel host1 anstelle von host1.intnet100), wird dnsmasq automatisch die intnet100-Domain anhängen, um den vollständigen FQDN zu bilden. Dies erleichtert die Namensauflösung, indem die Angabe der vollständigen Domäne vermieden wird.  
Insgesamt dienen diese Einstellungen dazu, die DNS-Namensauflösung in Ihrem lokalen Netzwerk zu verbessern, indem sie sicherstellen, dass nur gültige DNS-Anfragen beantwortet werden, und indem sie die interne Namensauflösung durch die Definition einer lokalen Domain erleichtern.  
