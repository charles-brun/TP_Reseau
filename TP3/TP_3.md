# TP3 : On va router des trucs

Au menu de ce TP, on va revoir un peu ARP et IP histoire de **se mettre en jambes dans un environnement avec des VMs**.

Puis on mettra en place **un routage simple, pour permettre à deux LANs de communiquer**.

![Reboot the router](./pics/reboot.jpeg)

## Sommaire

- [TP3 : On va router des trucs](#tp3--on-va-router-des-trucs)
  - [Sommaire](#sommaire)
  - [0. Prérequis](#0-prérequis)
  - [I. ARP](#i-arp)
    - [1. Echange ARP](#1-echange-arp)
    - [2. Analyse de trames](#2-analyse-de-trames)
  - [II. Routage](#ii-routage)
    - [1. Mise en place du routage](#1-mise-en-place-du-routage)
    - [2. Analyse de trames](#2-analyse-de-trames-1)
    - [3. Accès internet](#3-accès-internet)
  - [III. DHCP](#iii-dhcp)
    - [1. Mise en place du serveur DHCP](#1-mise-en-place-du-serveur-dhcp)
    - [2. Analyse de trames](#2-analyse-de-trames-2)

## 0. Prérequis

➜ Pour ce TP, on va se servir de VMs Rocky Linux. 1Go RAM c'est large large. Vous pouvez redescendre la mémoire vidéo aussi.  

➜ Vous aurez besoin de deux réseaux host-only dans VirtualBox :

- un premier réseau `10.3.1.0/24`
- le second `10.3.2.0/24`
- **vous devrez désactiver le DHCP de votre hyperviseur (VirtualBox) et définir les IPs de vos VMs de façon statique**

➜ Quelques paquets seront souvent nécessaires dans les TPs, il peut être bon de les installer dans la VM que vous clonez :

- de quoi avoir les commandes :
  - `dig`
  - `tcpdump`
  - `nmap`
  - `nc`
  - `python3`
  - `vim` peut être une bonne idée

➜ Les firewalls de vos VMs doivent **toujours** être actifs (et donc correctement configurés).

➜ **Si vous voyez le p'tit pote 🦈 c'est qu'il y a un PCAP à produire et à mettre dans votre dépôt git de rendu.**

## I. ARP

Première partie simple, on va avoir besoin de 2 VMs.

| Machine  | `10.3.1.0/24` |
|----------|---------------|
| `john`   | `10.3.1.11`   |
| `marcel` | `10.3.1.12`   |

```schema
   john               marcel
  ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     │
  └─────┘    └───┘    └─────┘
```

> Référez-vous au [mémo Réseau Rocky](../../cours/memo/rocky_network.md) pour connaître les commandes nécessaire à la réalisation de cette partie.

```
vim /etc/sysconfig/network-scripts/ifcfg-enp0s8
      DEVICE=enp0s8
      BOOTPROTO=none
      ONBOOT=yes
      PREFIX=24
      IPADDR=10.3.1.11
```

### 1. Echange ARP

🌞**Générer des requêtes ARP**

- effectuer un `ping` d'une machine à l'autre

```
ping 10.3.1.11
```

- observer les tables ARP des deux machines

```
ip n
```

- repérer l'adresse MAC de `john` dans la table ARP de `marcel` et vice-versa

```
MAC de john :       08:00:27:ad:79:e2
MAC de marcel :     08.00.27.09:2f:ce
```

- prouvez que l'info est correcte (que l'adresse MAC que vous voyez dans la table est bien celle de la machine correspondante)
  - une commande pour voir la MAC de `marcel` dans la table ARP de `john`

```
ip n show 10.3.1.2
```

  - et une commande pour afficher la MAC de `marcel`, depuis `marcel`

```
ip a
```

### 2. Analyse de trames

🌞**Analyse de trames**

- utilisez la commande `tcpdump` pour réaliser une capture de trame

```
tcpdump -w tp2_arp.pcapng
```

- videz vos tables ARP, sur les deux machines, puis effectuez un `ping`

```
ip -s -s neigh flush all
ping 10.3.1.11
```

🦈 **Capture réseau `tp2_arp.pcapng`** qui contient un ARP request et un ARP reply

![ARP](./tp2_arp.pcapng)

> **Si vous ne savez pas comment récupérer votre fichier `.pcapng`** sur votre hôte afin de l'ouvrir dans Wireshark, et me le livrer en rendu, demandez-moi.

```
sur machine hôte :  ssh cbrun@10.3.1.12
                    scp cbrun@10.3.1.12:/home/cbrun/tp2_arp.png ./
                    (cp via ssh de VM marcel vers dossier courant ./ de machine hote)
```

## II. Routage

Vous aurez besoin de 3 VMs pour cette partie. **Réutilisez les deux VMs précédentes.**

| Machine  | `10.3.1.0/24` | `10.3.2.0/24` |
|----------|---------------|---------------|
| `router` | `10.3.1.254`  | `10.3.2.254`  |
| `john`   | `10.3.1.11`   | no            |
| `marcel` | no            | `10.3.2.12`   |

> Je les appelés `marcel` et `john` PASKON EN A MAR des noms nuls en réseau 🌻

```schema
   john                router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └───┘    └─────┘    └───┘    └─────┘
```

### 1. Mise en place du routage

🌞**Activer le routage sur le noeud `router`**

> Cette étape est nécessaire car Rocky Linux c'est pas un OS dédié au routage par défaut. Ce n'est bien évidemment une opération qui n'est pas nécessaire sur un équipement routeur dédié comme du matériel Cisco.

```
vim /etc/sysctl.d/router.conf
      net.ipv4.ip_forward=1
```

🌞**Ajouter les routes statiques nécessaires pour que `john` et `marcel` puissent se `ping`**

- il faut ajouter une seule route des deux côtés

```
ajout du GATEWAY au fichier config, correspondant à l'adresse du routeur sur chaque réseau

vim /etc/sysconfig/network-scripts/ifcfg-enp0s8
      DEVICE=enp0s8
      BOOTPROTO=none
      ONBOOT=yes
      PREFIX=24
      IPADDR=10.3.1.11
      GATEWAY=10.3.1.254
```

- une fois les routes en place, vérifiez avec un `ping` que les deux machines peuvent se joindre

![THE SIZE](./pics/thesize.png)

### 2. Analyse de trames

🌞**Analyse des échanges ARP**

- videz les tables ARP des trois noeuds

```
ip -s -s neigh flush all
```

- effectuez un `ping` de `john` vers `marcel`

```
ping 10.3.2.12
```

- regardez les tables ARP des trois noeuds

```
table ARP         router            john              marcel
                  .1.11             .1.254            .2.254
                  .2.12
```

- essayez de déduire un peu les échanges ARP qui ont eu lieu

```
john (.1.11) a contacté sa passerelle (.1.254) correspondant au routeur, qui a transmis via sa carte du réseau 2 (.2.254) le paquet à marcel (.2.12) et l'inverse au retour du ping (marcel->routeur(2)->routeur(1)->john)
```

- répétez l'opération précédente (vider les tables, puis `ping`), en lançant `tcpdump` sur `marcel`
- **écrivez, dans l'ordre, les échanges ARP qui ont eu lieu, puis le ping et le pong, je veux TOUTES les trames** utiles pour l'échange

Par exemple (copiez-collez ce tableau ce sera le plus simple) :

| ordre | type trame  | IP source | MAC source              | IP destination | MAC destination            |
|-------|-------------|-----------|-------------------------|----------------|----------------------------|
| 1     | Requête ARP | x         | `passerelle R2`         | x              | Broadcast `FF:FF:FF:FF:FF` |
| 2     | Réponse ARP | x         | `marcel`                | x              | `passerelle R2`            |
| 3     | Ping        | .1.11     | `passerelle R2`         | .2.12          | `marcel`                   |
| 4     | Pong        | .2.12     | `marcel`                | .1.11          | `passerelle R2`            |
| 5     | Requête ARP | x         | `marcel`                | x              | Broadcast `FF:FF:FF:FF:FF` |
| 6     | Réponse ARP | x         | `passerelle R2`         | x              | `marcel`                   |

> Vous pourriez, par curiosité, lancer la capture sur `john` aussi, pour voir l'échange qu'il a effectué de son côté.

🦈 **Capture réseau `tp2_routage_marcel.pcapng`**

![Routage marcel](tp2_routage_marcel.pcapng)

### 3. Accès internet

🌞**Donnez un accès internet à vos machines**

- ajoutez une carte NAT en 3ème inteface sur le `router` pour qu'il ait un accès internet
- ajoutez une route par défaut à `john` et `marcel`
  - vérifiez que vous avez accès internet avec un `ping`
  - le `ping` doit être vers une IP, PAS un nom de domaine
- donnez leur aussi l'adresse d'un serveur DNS qu'ils peuvent utiliser
  - vérifiez que vous avez une résolution de noms qui fonctionne avec `dig`
  - puis avec un `ping` vers un nom de domaine

🌞**Analyse de trames**

- effectuez un `ping 8.8.8.8` depuis `john`
- capturez le ping depuis `john` avec `tcpdump`
- analysez un ping aller et le retour qui correspond et mettez dans un tableau :

| ordre | type trame | IP source          | MAC source              | IP destination | MAC destination |     |
|-------|------------|--------------------|-------------------------|----------------|-----------------|-----|
| 1     | ping       | `john` `10.3.1.12` | `john` `AA:BB:CC:DD:EE` | `8.8.8.8`      | `router`        |     |
| 2     | pong       | `8.8.8.8`          | `router`                | `john`         | `john`          | ... |

🦈 **Capture réseau `tp2_routage_internet.pcapng`**

![Routage internet](tp2_routage_internet.pcapng)

## III. DHCP

On reprend la config précédente, et on ajoutera à la fin de cette partie une 4ème machine pour effectuer des tests.

| Machine  | `10.3.1.0/24`              | `10.3.2.0/24` |
|----------|----------------------------|---------------|
| `router` | `10.3.1.254`               | `10.3.2.254`  |
| `john`   | `10.3.1.11`                | no            |
| `bob`    | oui mais pas d'IP statique | no            |
| `marcel` | no                         | `10.3.2.12`   |

```schema
   john               router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └─┬─┘    └─────┘    └───┘    └─────┘
   john        │
  ┌─────┐      │
  │     │      │
  │     ├──────┘
  └─────┘
```

### 1. Mise en place du serveur DHCP

🌞**Sur la machine `john`, vous installerez et configurerez un serveur DHCP** (go Google "rocky linux dhcp server").

- installation du serveur sur `john`

```
dnf install dhcp-server

vim /etc/dhcp/dhcpd.conf

      default-lease-time 900;
      max-lease-time 10800;
      ddns-update-style none;
      authoritative;
      subnet 10.3.1.0 netmask 255.255.255.0 {
        range 10.3.1.12 10.3.1.253;
        option routers 10.3.1.254;
        option subnet-mask 255.255.255.0;
        option domain-name-servers 1.1.1.1;

      }

firewall-cmd --permanent --add-port=67/udp

systemctl start dhcpd
```

- créer une machine `bob`
- faites lui récupérer une IP en DHCP à l'aide de votre serveur

```
vim /etc/sysconfig/network-scripts/ifcfg-enp0s8
      DEVICE=enp0s8

      BOOTPROTO=dhcp
      ONBOOT=yes
```

> Il est possible d'utilise la commande `dhclient` pour forcer à la main, depuis la ligne de commande, la demande d'une IP en DHCP, ou renouveler complètement l'échange DHCP (voir `dhclient -h` puis call me et/ou Google si besoin d'aide).

🌞**Améliorer la configuration du DHCP**

- ajoutez de la configuration à votre DHCP pour qu'il donne aux clients, en plus de leur IP :
  - une route par défaut
  - un serveur DNS à utiliser

```
vim /etc/dhcp/dhcpd.conf

      option routers 10.3.1.254;
      option domain-name-servers 1.1.1.1;

```

- récupérez de nouveau une IP en DHCP sur `marcel` pour tester :
  - `marcel` doit avoir une IP
    - vérifier avec une commande qu'il a récupéré son IP

```
ip a
```

    - vérifier qu'il peut `ping` sa passerelle

```
ping 10.3.2.254
```

  - il doit avoir une route par défaut
    - vérifier la présence de la route avec une commande

```
ip route show
```
    - vérifier que la route fonctionne avec un `ping` vers une IP

```
ping 8.8.8.8
```

  - il doit connaître l'adresse d'un serveur DNS pour avoir de la résolution de noms
    - vérifier avec la commande `dig` que ça fonctionne

```
dig ynov.com
```

    - vérifier un `ping` vers un nom de domaine

```
ping ynov.com
```

### 2. Analyse de trames

🌞**Analyse de trames**

- lancer une capture à l'aide de `tcpdump` afin de capturer un échange DHCP
- demander une nouvelle IP afin de générer un échange DHCP
- exportez le fichier `.pcapng`

🦈 **Capture réseau `tp2_dhcp.pcapng`**

![dhcp](tp2_dhcp.pcapng)