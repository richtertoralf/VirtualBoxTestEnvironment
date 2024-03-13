# VirtualBoxTestEnvironment
Testumgebungen in VirtualBox bauen
>Ab und zu entwickle ich Software mit der zwischen verschiedenen Maschinen Daten ausgetauscht werden müssen, also Software für verteilte Systeme.
>Für die Softwareentwicklung und das Testen dieser Software benötige ich Testumgebungen, die aus mehreren Computern und Routern bestehen.
>Bei längerfristigen Projekten nutze ich für das Ausrollen der Testumgebung Vagrant oder Ansible. Manchmal muss es aber auch mal schnell per Hand gemacht werden. Dazu im Folgenden paar Beispiele

## simpler virtueller Debian Router
### NIC planen und konfigurieren
### IP forwarding
Aktivieren der IP-Weiterleitung im Linux Kernel
### Konfigurieren der Routing-Software
Folgende drei Varianten nutze ich dazu: iptables, nftables oder Firewalld
#### iptables
#### nftables
#### Firewalld
## simpler DHCP-Server
