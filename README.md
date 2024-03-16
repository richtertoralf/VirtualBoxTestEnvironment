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

auto eth1
iface eth1 inet static
  address 192.168.100.1
  netmask 255.255.255.0

auto eth2
iface eth2 inet static
  address 192.168.200.1
  netmask 255.255.255.0
```

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
#### nftables
#### Firewalld
## simpler DHCP-Server
