# TP2 : Ethernet, IP, et ARP

Dans ce TP on va approfondir trois protocoles, qu'on a survolé jusqu'alors :

- **IPv4** *(Internet Protocol Version 4)* : gestion des adresses IP
  - on va aussi parler d'ICMP, de DHCP, bref de tous les potes d'IP quoi !
- **Ethernet** : gestion des adresses MAC
- **ARP** *(Address Resolution Protocol)* : permet de trouver l'adresse MAC de quelqu'un sur notre réseau dont on connaît l'adresse IP

![Seventh Day](./pics/tcpip.jpg)

# Sommaire

- [TP2 : Ethernet, IP, et ARP](#tp2--ethernet-ip-et-arp)
- [Sommaire](#sommaire)
- [0. Prérequis](#0-prérequis)
- [I. Setup IP](#i-setup-ip)
- [II. ARP my bro](#ii-arp-my-bro)
- [II.5 Interlude hackerzz](#ii5-interlude-hackerzz)
- [III. DHCP you too my brooo](#iii-dhcp-you-too-my-brooo)
- [IV. Avant-goût TCP et UDP](#iv-avant-goût-tcp-et-udp)

# 0. Prérequis

**Il vous faudra deux machines**, vous êtes libres :

- toujours possible de se connecter à deux avec un câble
- sinon, votre PC + une VM ça fait le taf, c'est pareil
  - je peux aider sur le setup, comme d'hab

> Je conseille à tous les gens qui n'ont pas de port RJ45 de go PC + VM pour faire vous-mêmes les manips, mais on fait au plus simple hein.

---

**Toutes les manipulations devront être effectuées depuis la ligne de commande.** Donc normalement, plus de screens.

**Pour Wireshark, c'est pareil,** NO SCREENS. La marche à suivre :

- vous capturez le trafic que vous avez à capturer
- vous stoppez la capture (bouton carré rouge en haut à gauche)
- vous sélectionner les paquets/trames intéressants (CTRL + clic)
- File > Export Specified Packets...
- dans le menu qui s'ouvre, cochez en bas "Selected packets only"
- sauvegardez, ça produit un fichier `.pcapng` (qu'on appelle communément "un ptit PCAP frer") que vous livrerez dans le dépôt git

**Si vous voyez le p'tit pote 🦈 c'est qu'il y a un PCAP à produire et à mettre dans votre dépôt git de rendu.**

# I. Setup IP

Le lab, il vous faut deux machine : 

- les deux machines doivent être connectées physiquement
- vous devez choisir vous-mêmes les IPs à attribuer sur les interfaces réseau, les contraintes :
  - IPs privées (évidemment n_n)
  - dans un réseau qui peut contenir au moins 38 adresses IP (il faut donc choisir un masque adapté)
  - oui c'est random, on s'exerce c'est tout, p'tit jog en se levant
  - le masque choisi doit être le plus grand possible (le plus proche de 32 possible) afin que le réseau soit le plus petit possible

🌞 **Mettez en place une configuration réseau fonctionnelle entre les deux machines**

- vous renseignerez dans le compte rendu :
  - les deux IPs choisies, en précisant le masque

```
192.168.0.1/26
192.168.0.2/26
```

  - l'adresse de réseau

```
192.168.0.0/26
```

  - l'adresse de broadcast

```
192.168.0.63/26
```

- vous renseignerez aussi les commandes utilisées pour définir les adresses IP *via* la ligne de commande

```
PC1 (windows):
        New-NetIPAddress -InterfaceAlias "VirtualBox Host-Only Network" -IPAddress 192.168.0.1 -PrefixLength "26"
PC2 (VM linux):
        ifconfig enp0s8 192.168.0.2 netmask 255.255.255.192
```

> Rappel : tout doit être fait *via* la ligne de commandes. Faites-vous du bien, et utilisez Powershell plutôt que l'antique cmd sous Windows svp.

🌞 **Prouvez que la connexion est fonctionnelle entre les deux machines**

- un `ping` suffit !

```
PS C:\Users\brunc> ping 192.168.0.2

Envoi d’une requête 'Ping'  192.168.0.2 avec 32 octets de données :
Réponse de 192.168.0.2 : octets=32 temps<1ms TTL=64
Réponse de 192.168.0.2 : octets=32 temps<1ms TTL=64
Réponse de 192.168.0.2 : octets=32 temps<1ms TTL=64
Réponse de 192.168.0.2 : octets=32 temps<1ms TTL=64

Statistiques Ping pour 192.168.0.2:
    Paquets : envoyés = 4, reçus = 4, perdus = 0 (perte 0%),
Durée approximative des boucles en millisecondes :
    Minimum = 0ms, Maximum = 0ms, Moyenne = 0ms
```

🌞 **Wireshark it**

- `ping` ça envoie des paquets de type ICMP (c'est pas de l'IP, c'est un de ses frères)
  - les paquets ICMP sont encapsulés dans des trames Ethernet, comme les paquets IP
  - il existe plusieurs types de paquets ICMP, qui servent à faire des trucs différents
- **déterminez, grâce à Wireshark, quel type de paquet ICMP est envoyé par `ping`**
  - pour le ping que vous envoyez

  ```
  request
  ```
  - et le pong que vous recevez en retour

```
reply
```

> Vous trouverez sur [la page Wikipedia de ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) un tableau qui répertorie tous les types ICMP et leur utilité

🦈 **PCAP qui contient les paquets ICMP qui vous ont permis d'identifier les types ICMP**

![ping](./pingpong.pcapng)

# II. ARP my bro

ARP permet, pour rappel, de résoudre la situation suivante :

- pour communiquer avec quelqu'un dans un LAN, il **FAUT** connaître son adresse MAC
- on admet un PC1 et un PC2 dans le même LAN :
  - PC1 veut joindre PC2
  - PC1 et PC2 ont une IP correctement définie
  - PC1 a besoin de connaître la MAC de PC2 pour lui envoyer des messages
  - **dans cette situation, PC1 va utilise le protocole ARP pour connaître la MAC de PC2**
  - une fois que PC1 connaît la mac de PC2, il l'enregistre dans sa **table ARP**

🌞 **Check the ARP table**

- utilisez une commande pour afficher votre table ARP

```
arp -a
```

- déterminez la MAC de votre binome depuis votre table ARP

```
Adresse Internet      Adresse physique        Type
192.168.0.2           08-00-27-fa-a4-7c       dynamique
```

- déterminez la MAC de la *gateway* de votre réseau 
  - celle de votre réseau physique, WiFi, genre YNOV, car il n'y en a pas dans votre ptit LAN

  ```
   192.168.0.254         00-24-d4-a4-55-34     dynamique
   (wi-fi domestique)
  ```
  - c'est juste pour vous faire manipuler un peu encore :)

> Il peut être utile de ré-effectuer des `ping` avant d'afficher la table ARP. En effet : les infos stockées dans la table ARP ne sont stockées que temporairement. Ce laps de temps est de l'ordre de ~60 secondes sur la plupart de nos machines.

🌞 **Manipuler la table ARP**

- utilisez une commande pour vider votre table ARP

```
arp -d
```

- prouvez que ça fonctionne en l'affichant et en constatant les changements

```
Interface : 192.168.0.1 --- 0xb
  Adresse Internet      Adresse physique      Type
  224.0.0.22            01-00-5e-00-00-16     statique
```

- ré-effectuez des pings, et constatez la ré-apparition des données dans la table ARP

```
Interface : 192.168.0.1 --- 0xb
  Adresse Internet      Adresse physique      Type
  192.168.0.2           08-00-27-fa-a4-7c     dynamique
  224.0.0.22            01-00-5e-00-00-16     statique
  239.255.255.250       01-00-5e-7f-ff-fa     statique
```

> Les échanges ARP sont effectuées automatiquement par votre machine lorsqu'elle essaie de joindre une machine sur le même LAN qu'elle. Si la MAC du destinataire n'est pas déjà dans la table ARP, alors un échange ARP sera déclenché.

🌞 **Wireshark it**

- vous savez maintenant comment forcer un échange ARP : il sufit de vider la table ARP et tenter de contacter quelqu'un, l'échange ARP se fait automatiquement
- mettez en évidence les deux trames ARP échangées lorsque vous essayez de contacter quelqu'un pour la "première" fois

```
1	0.000000	0a:00:27:00:00:0b	Broadcast	ARP	42	Who has 192.168.0.2? Tell 192.168.0.1
2	0.000191	PcsCompu_fa:a4:7c	0a:00:27:00:00:0b	ARP	60	192.168.0.2 is at 08:00:27:fa:a4:7c
```

  - déterminez, pour les deux trames, les adresses source et destination

```
      src                                 dest
1     0a:00:27:00:00:0b                   ff:ff:ff:ff:ff:ff
2     08:00:27:a4:7c                      0a:00:27:00:00:0b
```
  - déterminez à quoi correspond chacune de ces adresses

```
      src                                 dest
1     PC1                                 broadcast
2     PC2                                 PC1
```

🦈 **PCAP qui contient les trames ARP**

> L'échange ARP est constitué de deux trames : un ARP broadcast et un ARP reply.

![échange ARP](./arp.pcapng)

# II.5 Interlude hackerzz

**Chose promise chose due, on va voir les bases de l'usurpation d'identité en réseau : on va parler d'*ARP poisoning*.**

> On peut aussi trouver *ARP cache poisoning* ou encore *ARP spoofing*, ça désigne la même chose.

Le principe est simple : on va "empoisonner" la table ARP de quelqu'un d'autre.  
Plus concrètement, on va essayer d'introduire des fausses informations dans la table ARP de quelqu'un d'autre.

Entre introduire des fausses infos et usurper l'identité de quelqu'un il n'y a qu'un pas hihi.

---

Le principe de l'attaque :

- on admet Alice, Bob et Eve, tous dans un LAN, chacun leur PC
- leur configuration IP est ok, tout va bien dans le meilleur des mondes
- **Eve 'lé pa jonti** *(ou juste un agent de la CIA)* : elle aimerait s'immiscer dans les conversations de Alice et Bob
  - pour ce faire, Eve va empoisonner la table ARP de Bob, pour se faire passer pour Alice
  - elle va aussi empoisonner la table ARP d'Alice, pour se faire passer pour Bob
  - ainsi, tous les messages que s'envoient Alice et Bob seront en réalité envoyés à Eve

La place de ARP dans tout ça :

- ARP est un principe de question -> réponse (broadcast -> *reply*)
- IL SE TROUVE qu'on peut envoyer des *reply* à quelqu'un qui n'a rien demandé :)
- il faut donc simplement envoyer :
  - une trame ARP reply à Alice qui dit "l'IP de Bob se trouve à la MAC de Eve" (IP B -> MAC E)
  - une trame ARP reply à Bob qui dit "l'IP de Alice se trouve à la MAC de Eve" (IP A -> MAC E)
- ha ouais, et pour être sûr que ça reste en place, il faut SPAM sa mum, genre 1 reply chacun toutes les secondes ou truc du genre
  - bah ui ! Sinon on risque que la table ARP d'Alice ou Bob se vide naturellement, et que l'échange ARP normal survienne
  - aussi, c'est un truc possible, mais pas normal dans cette utilisation là, donc des fois bon, ça chie, DONC ON SPAM

![Am I ?](./pics/arp_snif.jpg)

---

J'peux vous aider à le mettre en place, mais **vous le faites uniquement dans un cadre privé, chez vous, ou avec des VMs**

**Je vous conseille 3 machines Linux**, Alice Bob et Eve. La commande `[arping](https://sandilands.info/sgordon/arp-spoofing-on-wired-lan)` pourra vous carry : elle permet d'envoyer manuellement des trames ARP avec le contenu de votre choix.

GLHF.

# III. DHCP you too my brooo

![YOU GET AN IP](./pics/dhcp.jpg)

*DHCP* pour *Dynamic Host Configuration Protocol* est notre p'tit pote qui nous file des IP quand on arrive dans un réseau, parce que c'est chiant de le faire à la main :)

Quand on arrive dans un réseau, notre PC contacte un serveur DHCP, et récupère généralement 3 infos :

- **1.** une IP à utiliser
- **2.** l'adresse IP de la passerelle du réseau
- **3.** l'adresse d'un serveur DNS joignable depuis ce réseau

L'échange DHCP consiste en 4 trames : DORA, que je vous laisse google vous-mêmes : D

🌞 **Wireshark it**

- identifiez les 4 trames DHCP lors d'un échange DHCP
  - mettez en évidence les adresses source et destination de chaque trame

  ```
  Trame                         Source                          Destination
  Discover                      0.0.0.0 (PC sans IP assignée)   255.255.255.255 (broadcast recherche du server DHCP)
  Offer                         10.33.19.254 (server DHCP)      10.33.18.109 (PC avec IP proposée)
  Request                       0.0.0.0                         255.255.255.255
  Acknowledge                   10.33.19.254                    10.33.18.109
  ```
  
- identifiez dans ces 4 trames les informations **1**, **2** et **3** dont on a parlé juste au dessus

```
1 : 10.33.18.109
2 : 10.33.19.254
3 : 8.8.8.8
```

🦈 **PCAP qui contient l'échange DORA**

![DORA](./DORA.pcapng)

> **Soucis** : l'échange DHCP ne se produit qu'à la première connexion. **Pour forcer un échange DHCP**, ça dépend de votre OS. Sur **GNU/Linux**, avec `dhclient` ça se fait bien. Sur **Windows**, le plus simple reste de définir une IP statique pourrie sur la carte réseau, se déconnecter du réseau, remettre en DHCP, se reconnecter au réseau. Sur **MacOS**, je connais peu mais Internet dit qu'c'est po si compliqué, appelez moi si besoin.

# IV. Avant-goût TCP et UDP

TCP et UDP ce sont les deux protocoles qui utilisent des ports. Si on veut accéder à un service, sur un serveur, comme un site web :

- il faut pouvoir joindre en terme d'IP le correspondant
  - on teste que ça fonctionne avec un `ping` généralement
- il faut que le serveur fasse tourner un programme qu'on appelle "service" ou "serveur"
  - le service "écoute" sur un port TCP ou UDP : il attend la connexion d'un client
- le client **connaît par avance** le port TCP ou UDP sur lequel le service écoute
- en utilisant l'IP et le port, il peut se connecter au service en utilisant un moyen adapté :
  - un navigateur web pour un site web
  - un `ncat` pour se connecter à un autre `ncat`
  - et plein d'autres, **de façon générale on parle d'un client, et d'un serveur**

---

🌞 **Wireshark it**

- déterminez à quelle IP et quel port votre PC se connecte quand vous regardez une vidéo Youtube
  - il sera sûrement plus simple de repérer le trafic Youtube en fermant tous les autres onglets et toutes les autres applications utilisant du réseau

  ```
                        IP                    PORT
  Src (mon PC)          10.33.18.109          59793
  Dest (youtube)        216.58.204.110        443
  ```

🦈 **PCAP qui contient un extrait de l'échange qui vous a permis d'identifier les infos**

![video youtube](./vid_yt.pcapng)
