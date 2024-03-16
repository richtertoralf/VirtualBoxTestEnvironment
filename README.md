# VirtualBoxTestEnvironment
Testumgebungen in VirtualBox bauen
>Ab und zu entwickle ich Software mit der zwischen verschiedenen Maschinen Daten ausgetauscht werden soll, also Software für verteilte Systeme.
>Für die Softwareentwicklung und das Testen dieser Software benötige ich Testumgebungen, die oft aus mehreren Computern und Routern bestehen.
>Bei längerfristigen Projekten nutze ich für das Ausrollen der Testumgebung Vagrant oder Ansible. Manchmal muss es aber auch mal schnell per Hand gemacht werden. Dazu im Folgenden paar Beispiele

## simpler virtueller Debian Router
![simple Testumgebung / Router mit zwei Subnetzen und DHCP](https://github.com/richtertoralf/VirtualBoxTestEnvironment/blob/24660940c16c5e3eb00373b97982ac7ac37586ea/VB_TestEnvironment_01.jpg)
### NIC planen und konfigurieren
In Debian wird die Datei `/etc/network/interfaces` verwendet, um Netzwerkschnittstellen zu konfigurieren.
```
auto eth0
iface eth0 inet dhcp
iface eth0 inet6 auto

auto eth1
iface eth1 inet static
    address 192.168.100.1
    netmask 255.255.255.0

iface eth1 inet6 static
    address fd00::192:168:100::1
    netmask 64

auto eth2
iface eth2 inet static
    address 192.168.200.1
    netmask 255.255.255.0

iface eth2 inet6 static
    address fd00::192:168:200::1
    netmask 64
```
eth0: Dies ist die Schnittstelle, die mit dem Internet verbunden ist und eine IPv4-Adresse über DHCP bezieht. Die IPv6-Adresse wird automatisch konfiguriert.

eth1 und eth2: Dies sind die internen Schnittstellen, die mit den internen Netzwerken verbunden sind. Sie sind statisch konfiguriert sowohl für IPv4 als auch für IPv6.

Die IPv4-Adressen 192.168.100.1 und 192.168.200.1 sind die Gateway-Adressen für die internen Netzwerke, und die entsprechenden IPv6-Adressen fd00::192:168:100::1 und fd00::192:168:200::1 sind die ULAs für die internen Netzwerke.

### IP forwarding
Aktivieren der IP-Weiterleitung im Linux Kernel  
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
#### Firewalld
## simpler DHCP-Server
