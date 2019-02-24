# TP1 - Réseau
## 1/ Mise en place
### Configuration de VirtualBox

On créer 2 nouveaux réseaux. On clone la VM faites en cours et on lui attribue ces deux nouveaux réseaux.

net1 : 10.1.1.0/24
Combien d'adresses disponibles ? 
2^8 -2 = 254 
Il y a 254 adresses disponibles dans un réseau /24.

net2 : 10.1.2.0/30
Combien d'adresses disponibles ?	
2^2 - 2 = 2
Il y a 2 adresses disponibles dans un réseau /24.
Quelle est l'utilité d'un /30 ?
L'utilité d'un /30 est de maitriser le nombre d'adresses disponibles dans un réseau (2 adresses dans ce cas).

### Configuration de la VM

 - SELinx
	 - Déjà effectué dans le patron
 - Installation de certains paquets réseaux
	 - Déjà effectué dans le patron
 - Définition d'IP statique sur les deux cartes host-only
	- `ip a` : on vérifie que nos cartes enp0s3 (NAT), enp0s8 et enp0s9 (host-only) sont bien présentes.
	- `sudo nano /etc/sysconfig/network-scripts/ifcfg-enp0s8`
	- `cat /etc/sysconfig/network-scripts/ifcfg-enp0s8` nous retourne : `TYPE=Ethernet BOOTPROTO=static IPADDR=10.1.1.2 NETMASK=255.255.255.0 NAME=enp0s8 DEVICE=enp0S8 ONBOOT=yes`
	- Même chose avec enp0s9 à la place de enp0s8 avec `IPADDR=10.1.2.2` et `NETMASK=255.255.255.252`
	- `sudo ifdown <nom>`, puis `sudo ifup <nom>` pour les deux cartes modifiés (enp0s8 et enp0s9)
- Connexion en SSH
	- On ouvre Putty et on utilise l'IP `10.1.1.2`
- Définition d'un nom de domaine
	- `sudo nano /etc/hostname`
	- on écrit : `client1.tp1.b2`
- Compléter le fichier hosts de la VM
	- `sudo nano /etc/hosts`
	- `cat /etc/hosts` : `127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost4 localhost4.localdomain4
10.1.1.2    client1 client1.tp1.b2
10.1.2.2    client1 client1.tp1.b2
10.1.1.1    monpc monpc.chezmoi
10.1.2.1    monpc monpc.chezmoi
`
- On s'assure que toutes les cartes réseaux fonctionnent
	- NAT : `curl google.com` renvoie une erreur 301 (la page a été déplace, il faut aller sur HTTPS), on utilise donc `curl -L google.com` pour ne plus avoir cette erreur.
	- Hosts-only : on vérifie que le ping fonctionne bien `ping 10.1.1.1`et `ping 10.1.2.1`

### Basics

#### Routes
#### Table ARP

1 - On affiche la table arp avec : `ip n s`

	10.1.2.1 dev enp0s9 lladdr 0a:00:27:00:00:44 STALE
Notre VM connait notre hôte via l'IP 10.1.2.1 (carte réseau enp0s9).

	10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:40 DELAY
Notre VM connait notre hôte via l'IP 10.1.1.1 (carte réseau enp0s8).

	10.0.2.2 dev enp0s3 lladdr 52:54:00:12:35:02 STALE
Il s'agit de notre route par défaut (notre NAT) qui nous permet notamment de nous connecter à internet.

2 - On vide la table ARP

	sudo ip neigh flush all

On vérifie que la table a bien été vidé avec la commande `ip n s`, cependant nous avons un résultat car nous sommes connecté à la VM en SSH:  

	10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:40 REACHABLE

3 - Effectuer une requête simple vers l'hôte

On `ping 10.1.2.1`, puis on affiche la table arp avec `ip n s`.

	10.1.2.1 dev enp0s9 lladdr 0a:00:27:00:00:44 REACHABLE
	10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:40 REACHABLE

Le ping a détecté que `10.1.2.1` était un "voisin" de notre VM (via la carte réseau enp0s9). 

## 2/ Communication simple entre deux machines

### Mise en place

On clone notre VM de base. On lui donne une carte host-only sur `10.1.1.3` et une carte NAT.
On refait toute la phase de configuration faites précédemment (on adapte à notre cas). 
On n'oublie pas de modifier le fichier `/etc/hosts` de notre première VM.

### Basics

1 - On vide les tables ARP des deux VMs avec `sudo ip neigh flush all`

2 - On ping client2 depuis client1 : `ping -c 4 client2`

3 - On affiche les tables ARP des VMs avec la commande `ip n s`:

client1 : 
	
	10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:40 REACHABLE
	10.1.1.3 dev enp0s8 lladdr 08:00:27:f8:2b:de REACHABLE
En `10.1.1.1` on a la connexion SSH. Et en `10.1.1.3` client2.

client2 : 
	
	10.1.1.2 dev enp0s8 lladdr 08:00:27:61:f0:ae REACHABLE
	10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:40 DELAY

En `10.1.1.1` on a la connexion SSH. Et en `10.1.1.2` client1.

4 - Capture réseau
On se connecte en SSH sur la VM2 en `10.1.1.3`.
On se connecte en SSH sur la VM1 en `10.1.2.2`.

On vide les tables ARP (`sudo ip neigh flush all`).
Puis depuis client1 : `sudo tcpdump -i enp0s8 -w ping-2.pcap`.
Et on ping finalement depuis client 2 : `ping -c 4 10.1.1.2`.

Dans client 1 on fait **CTRL+C** et on obtient les lignes suivantes: 

	10 packets captured
	10 packets received by filter
	0 packets dropped by kernel
	
Sur les 10 paquets capturés il y a:
- 4 pings
- 4 pongs
- 2 paquets d'échange ARP

5 - Analyse WireShark

On récupère ping-2.pcap : 

	PS C:\Users\Florian> scp florian@10.1.1.2:/home/florian/ping-2.pcap .\Desktop\

On ouvre le fichier avec WireShark et on obtient 2 lignes oranges pour les paquets ARP et 8 lignes roses pour le protocole ICMP (les 4 pings et 4 pongs).

#### UDP

On ouvre le port UDP 8888 sur client1, puis on écoute ce dernier : 

	sudo firewall-cmd --add-port=8888/udp --permanent
	sudo firewall-cmd --reload
	nc -u -l 8888
Puis on se connecte sur le port UDP 8888 du client1 depuis le client2 : 
	
	nc -u client1 8888

Pour voir la connexion établie on utilise `ss -unp`.

client1 : 

	Recv-Q Send-Q                    Local Address:Port                                   Peer Address:Port
	0      0                              10.1.1.2:8888                                       10.1.1.3:45629               users:(("nc",pid=1623,fd=4))
	
client2 : 

	Recv-Q Send-Q                    Local Address:Port                                   Peer Address:Port
	0      0                              10.1.1.3:45629                                       10.1.1.2:8888                users:(("nc",pid=1486,fd=3))

On remarque la connexion entre les deux VMs sur le

On analyse la connexion avec tcpdump (pour capturer les messages/divers paquets) : 

    sudo tcpdump -i enp0s8 -w nc-udp.pcap

On s'échange quelques messages puis on ferme la connexion `netcat` et  `tcpdump`.

On récupère l'échange sur l'hôte avant de l'analyser sur WireShark : 

    scp florian@10.1.1.2:/home/florian/nc-udp.pcap .\Desktop\

Sur WireShark on trouve la présence de :
- paquets ARP
- Et les messages dans des paquets UDP, on constate qu'il n'y a pas de tunnel ni de “3-way handshake".

#### TCP

On ouvre le port TCP 8888 sur client1, puis on écoute ce dernier : 

	sudo firewall-cmd --add-port=8888/tcp --permanent
	sudo firewall-cmd --reload
	nc -l 8888
Puis on se connecte sur le port TCP 8888 du client1 depuis le client2 : 
	
	nc client1 8888

Pour voir la connexion établie on utilise `ss -tnp`.
client1 : 

	State      Recv-Q Send-Q Local Address:Port               Peer Address:Port     
	ESTAB      0      0      10.1.1.2:22                 10.1.1.1:60237             
	ESTAB      0      0      10.1.1.2:8888               10.1.1.3:47902               users:(("nc",pid=13913,fd=5))
	ESTAB      0      0      10.1.2.2:22                 10.1.2.1:60132             

client2:

	State      Recv-Q Send-Q Local Address:Port               Peer Address:Port     
	ESTAB      0      0      10.1.1.3:22                 10.1.1.1:60052             
	ESTAB      0      0      10.1.1.3:47902              10.1.1.2:8888                users:(("nc",pid=4087,fd=3))
	ESTAB      0      0      10.1.1.3:22                 10.1.1.1:60242             

On remarque que les connexions établies sur les deux VMs sont : 
- 2 connexions SSH
- La connexion entre les deux VMs sur la machine `10.1.1.2:8888`

On analyse la connexion avec tcpdump (pour capturer les messages/divers paquets) : 

    sudo tcpdump -i enp0s8 -w nc-tcp.pcap

On établit la connexion avec `netcat`, on s'échange quelques messages puis on ferme la connexion `netcat` avant d'arrêter `tcpdump`.

On récupère l'échange sur l'hôte avant de l'analyser sur WireShark : 

    scp florian@10.1.1.2:/home/florian/nc-tcp.pcap .\Desktop\

Sur WireShark on trouve la présence de :
- 2 paquets ARP
- 2 paquets SYNC pour la synchronisation des deux machines
- A chaque message on retrouve : le message, un "PUSH" et un "ACK" (acknowledgement) de la part de l'expéditeur et un "ACK" du destinataire. Il s'agit du "3-way handshake TCP".
- 2 paquets de FIN de communication avec les paquets "ACK" associés

## 3/ Routage statique simple

On commence par transformer le client1 en routeur, puis on sur client2 on ajoute une route statique vers net2.

client1 : 
	
	sudo sysctl -w net.ipv4.ip_forward=1

client2 : 

	sudo ip route add 10.1.2.0/30 via 10.1.1.2 dev enp0s9

Ensuite on vérifie le bon fonctionnement du routage à l'aide d'un `ping` puis d'un `traceroute` (client2) : 

	$ ping 10.1.2.1
	PING 10.1.2.1 (10.1.2.1) 56(84) bytes of data.
	64 bytes from 10.1.2.1: icmp_seq=1 ttl=127 time=0.516 ms
	64 bytes from 10.1.2.1: icmp_seq=2 ttl=127 time=0.657 ms
	64 bytes from 10.1.2.1: icmp_seq=3 ttl=127 time=0.683 ms
	^C
	--- 10.1.2.1 ping statistics ---
	3 packets transmitted, 3 received, 0% packet loss, time 2076ms
	rtt min/avg/max/mdev = 0.516/0.625/0.683/0.082 ms

	$ traceroute 10.1.2.1
	traceroute to 10.1.2.1 (10.1.2.1), 30 hops max, 60 byte packets
	1  gateway (10.0.2.2)  0.217 ms  0.195 ms  0.124 ms
	2  10.1.2.1 (10.1.2.1)  0.321 ms  0.253 ms  0.364 ms
