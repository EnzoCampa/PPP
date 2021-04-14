 # M3103 - Compte rendue PPP
 
 
Un serveur PPP peut avoir plusieur client connecter en même temps.
Il attribut 1 ID à chaque client, il peut avoit 65535 client car l’ID est coder sur 16bit.
La trame PADO porte l’ac-name du serveur.

## 1 - Les paquets utiliser

Grace aux outils apt search et apt shows afin de regarder les paquets installer avec ppp.


|           | Client                                              | Serveur                                                |
| --------- | --------------------------------------------------- | ------------------------------------------------------ |
| PPP       | Utile à pppoe et pppoeconf (automatique)            | Utile à ppoe et pppoeconf (automatique)                |
| pppoe     | Dépend de ppp (automatique)                         | Dépend de ppp on l'installe sur le serveur (manuelle ) | 
| pppoeconf | Dépend de pppoe On installe sur le client(manuelle) | On ne  l'installe pas sur le serveur !                 |


## 2 - Mise en place d’un client et d’un serveur

### Etape 1 : La liaison 

En premier lieu, on va lancer le serveur et le client.

La commande pour le lancer est : 

```bash=
pppoe-server -C enzo -I enp0s3
```

Puis on lance sur le client la commande : 

```bash= 
pppoeconf
```

En lancant wireshark on peut voir des paquets : 

PADI = Ce paquet est envoyé par le client pour trouver le serveur et demander une adresse ip.
PADO = Paquet envoyé par le serveur pour répondre à la demande d’un client.
PADR = Demande la confirmation de l'adresse ip reçus.
PADS = Approuve la confirmation.
PADT(2) = Met fin à la connexion, il y a un paquet émis par le client et un autre émis par le serveur.

Etant donné que l'on voie des PADT sans avoir clos la connectio, cela prouve que la connection ne s'est pas bien passé.

### Etape 2 : adressage dynamique 


```bash=
pppoe-server -C enzo -L 192.168.1.150 -p /etc/ppp/ipaddress_pool -I enp0s3
```

L'option -L permet de donner l'adresse IP du serveur. Ici 192.168.1.150
Pour le pool d'adresse IP, on les rentres dans le fichiers /etc/ppp/ipaddress_pool. Il va voir le fichier qui contient la plage d'adresse ip que le serveur pourra délivrer.

Dans le fichier ipaddress_pool on configure la plage d’adresse ip: 

```bash=
192.168.1.151-199
```

### Etape 3 : l’authentification

```bash=
pppoe-server -C enzo -L 192.168.1.150 -p /etc/ppp/ipaddress_pool -I enp0s3
```

Ici, on autorise dans le fichier option l'identification avec pap-secret et on les modifies pour mettres le ou les identidiants et le ou les mots de passe.

## 3 - Communication

Plusieurs ping ont été fait sur des interfaces différentes.

- L’interface PPP du serveur
- L’interface eno1 du serveur
- La passerelle

J'ai constaté que l'on ne pouvait pas ping la passerelle mais l'interface ppp et eno était pinguable. Donc, voici le shéma qui explique le chemin que prend le paquet.

![](https://i.imgur.com/5MvHJY9.png)


Du coup il nous faut configurer le forwarding il se fera en deux commandes : 

```bash=
echo 1 > /proc/sys/net/ipv4/ip_forward
echo 1 > /proc/sys/net/ipv4/conf/eno/proxy_arp
```

## 4 - Analyse de trames

Ci dessous on voit la capture de la trame PADI, nous remarquons que le session ID est 0x0000.

Initiation (PADI)
Frame 3: 24 bytes on wire (192 bits), 24 bytes captured (192 bits) on interface 0
Ethernet II, Src: Dell_b8:d4:44 (b8:ca:3a:b8:d4:44), Dst: Broadcast (ff:ff:ff:ff:ff:ff)
PPP-over-Ethernet Discovery
 0001 .... = Version: 1
 .... 0001 = Type: 1
 Code: Active Discovery Initiation (PADI) (0x09)
 Session ID: 0x0000
 Payload Length: 4
 PPPoE Tags
 
 Ci dessous on à la trame PADO, nous remarquons que le session ID n’a pas bouger et qu’il y a des information suplémentaire dans PPPoE tags on trouve un AC-name et AC-cookie.

Offer (PADO) AC-Name='232-22'
Frame 4: 60 bytes on wire (480 bits), 60 bytes captured (480 bits) on interface 0
Ethernet II, Src: Luxshare_02:b9:c0 (3c:18:a0:02:b9:c0), Dst: Dell_b8:d4:44 (b8:ca:3a:b8:d4:44)
PPP-over-Ethernet Discovery
 0001 .... = Version: 1
 .... 0001 = Type: 1
 Code: Active Discovery Offer (PADO) (0x07)
 Session ID: 0x0000
 Payload Length: 38
 PPPoE Tags
 AC-Name: 232-22
 AC-Cookie: 85686fad101713e17cb4e92c16a1f1e4b83a0000
 
 
 Ci dessous on à la trame PADR, nous remarquons que le session ID n’a pas bouger et qu’il y a les
même information dans le AC-Cookie tags que dans le PADO .

Request (PADR)
Frame 5: 48 bytes on wire (384 bits), 48 bytes captured (384 bits) on interface 0
Ethernet II, Src: Dell_b8:d4:44 (b8:ca:3a:b8:d4:44), Dst: Luxshare_02:b9:c0 (3c:18:a0:02:b9:c0)
PPP-over-Ethernet Discovery
 0001 .... = Version: 1
 .... 0001 = Type: 1
 Code: Active Discovery Request (PADR) (0x19)
 Session ID: 0x0000
 Payload Length: 28
 PPPoE Tags
 AC-Cookie: 85686fad101713e17cb4e92c16a1f1e4b83a0000

 
 Ci dessous on à la trame PADS, nous remarquons que le session ID est 0x000c et qu’i n’y a plus d’information dans le PPPoE tags.
 
Session-confirmation (PADS)
Frame 6: 60 bytes on wire (480 bits), 60 bytes captured (480 bits) on interface 0
Ethernet II, Src: Luxshare_02:b9:c0 (3c:18:a0:02:b9:c0), Dst: Dell_b8:d4:44 (b8:ca:3a:b8:d4:44)
PPP-over-Ethernet Discovery
 0001 .... = Version: 1
 .... 0001 = Type: 1
 Code: Active Discovery Session-confirmation (PADS) (0x65)
 Session ID: 0x000c
 Payload Length: 4
 PPPoE Tags
 
 
 Ci dessous on à la trame PADT, nous remarquons que le session ID est 0x000c et qu’i n’y a plus d’information dans le PPPoE tags

Terminate (PADT)
Frame 7: 44 bytes on wire (352 bits), 44 bytes captured (352 bits) on interface 0
Ethernet II, Src: Dell_b8:d4:44 (b8:ca:3a:b8:d4:44), Dst: Luxshare_02:b9:c0 (3c:18:a0:02:b9:c0)
PPP-over-Ethernet Discovery
 0001 .... = Version: 1
 .... 0001 = Type: 1
 Code: Active Discovery Terminate (PADT) (0xa7)
 Session ID: 0x000c
 Payload Length: 24
 PPPoE Tags
 AC-Cookie: 85686fad101713e17cb4e92c16a1f1e4b83a0000
 
 On remarque que l’ID de session ne change qu’a partir de la trame PADS.

## 5 - Multiclient test

Sur le ping suivant on voit que je ping le client

root@232-22:~$ ping 192.168.1.102
PING 192.168.1.102 (192.168.1.102) 56(84) bytes of data.
64 bytes from 192.168.1.102: icmp_seq=1 ttl=64 time=1.14 ms
64 bytes from 192.168.1.102: icmp_seq=2 ttl=64 time=1.05 ms
^C
--- 10.214.13.50 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 1.046/1.091/1.137/0.056 ms


Sur le ping suivant on voit que je ping 1.1.1.1

root@232-22:~$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=54 time=117 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=54 time=27.0 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=54 time=41.9 ms
^C
--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 27.028/61.902/116.783/39.279 ms
test@232-22:~$

### Client Windows


Pour configuré le client Windows nous avons suivi le tuto suivant : 

https://tp-link.com/fr/support/faq/339/

Le client windows a bien pu se connecter au serveur, ip=192.168.1.116
On retrouve sur le client Windows les même trame que sur un client Linux.
On remarque juste l’apparition d’un Host-Uniq sur chaque trame dans le pppoe tags

exemple de padi 

Frame 81: 60 bytes on wire (480 bits), 60 bytes captured (480 bits) on interface 0
Ethernet II, Src: PcsCompu_b6:9f:d5 (08:00:27:b6:9f:d5), Dst: Broadcast (ff:ff:ff:ff:ff:ff)
PPP-over-Ethernet Discovery
0001 … = Version: 1
… 0001 = Type: 1
Code: Active Discovery Initiation (PADI) (0x09)
Session ID: 0x0000
Payload Length: 20
PPPoE Tags
Host-Uniq: 050000000000000009000000