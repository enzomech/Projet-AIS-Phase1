# 02-Configuration réseau

On se concentre ici sur la configuration du réseau sur notre serveur ubuntu fraîchement installé.

1. [Nommer les intefaces réseaux](#nommer-les-intefaces-réseaux)
2. [Aggrégation des liens](#aggrégation-des-liens)


---


## Nommer les intefaces réseaux

### Création de règle Udev

Il est important de spécifier la bonne interface et la bonne adresse MAC pour nommer correctement nos interfaces, on peut obtenir ces informations avec la commande ```ip link show```.

Après avoir noté nos adresses MAC, on créer ou édite le fichier suivant : ```/etc/udev/rules.d/10-network-interface-names.rules```

On y rajoute cette ligne en modifiant les informations requises :

```SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="<ADDRESSE MAC>", NAME="<NOM DE L'INTERFACE>"```

Après modification, mon fichier ressemble à ça :

```
# Interface publique, configurée en NAT
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="00:0c:29:f5:2d:2e", NAME="public0"

# Interfaces privées, configurées en hôtes privées, utilisées en aggregat
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="00:0c:29:f5:2d:38", NAME="private0"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="00:0c:29:f5:2d:42", NAME="private1"
```

### Désactiver le renommage automatique par systemd (facultatif mais recommandé)

Il faut éditer le fichier GRUB : 
```/etc/default/grub```

On cherche la ligne :
```GRUB_CMDLINE_LINUX=""```

Et on la modifie ainsi :
```GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"```

Puis on met à jour grub :
```update-grub```

On le laisse générer la nouvelle configuration, puis on redémarre pour vérifier nos interfaces avec ```ip link show```

### Adapter la configuration netplan :

Il faut aussi modifier la configuration de netplan (ou autre gestionnaire) pour utiliser les nouveaux noms d'interface

Dans le fichier ```/etc/netplan/*.yaml``` il faut entrer notre nouvelle configuration, voici la mienne :

```
network:
    version: 2
    ethernets:
        public0:
            dhcp4: false
            addresses: [192.168.150.10/24]
        private0:
            dhcp4: false
        private1:
            dhcp4: false
```

On sauvegarde et applique la nouvelle configuration avec ```netplan apply``` (ou redémmarage du réseau).


### Problème de configuration netplan


#### Source du problème

Il ne faut pas oublier de suivre la procédure pour conserver les règles dans le fichier de configuration netplan (celle indiquée en commentaire en haut du fichier de config netplan) :

```
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
```


#### Configuration spécifiée

Voici un guide rapide pour cette procédure, on commence par créer le fichier spécifié :

```touch /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg```

On y glisse la configuration spécifiée également :

```network: {config: disabled}```


#### Configuration recommandée

Pour une configuration plus propre (notamment par soucis de clarté avec la configuration auto générée), il est recommandé de créer son propre fichier netplan persistant.

On supprime donc l'ancienne configuration auto générée :
```rm -f /etc/netplan/50-cloud-init.yaml```

Puis on créer la notre, par exemple :
```/etc/netplan/01-netcfg.yaml```

Ce fichier étant sensible, on va modifier les permissions, par défaut trop ouvertes, de ce nouveau fichier. 
On commence par changer l'utilisateur et le groupe utilisateur propriétaire du fichier si ce n'est pas déjà celui du root :

```chown root:root /etc/netplan/01-netcfg.yaml```

Ensuite on change les droits d'accès au fichier, pour rester conforme aux standards Ubuntu, on peut utiliser un droit de 644 (lecture pour tous, écriture uniquement pour root), c'est suffisant et plus pratique pour un serveur non critique ou pas encore en production, mais on peut aussi utiliser un droit de 600 (lecture/écriture pour root uniquement) pour plus de sécurité. On utilise la commande suivante :

```chmod 644 /etc/netplan/01-netcfg.yaml```

On peut finalement vérifier les droits, propriétaires et groupes d’un fichier avec :

```ls -l /etc/netplan/01-netcfg.yaml```

On oublie pas de glisser notre configuration dans le nouveau fichier, puis on applique notre netplan comme fait précédement, on peut redémarrer la machine pour voir si la configuration a été persistante, puis vérifier avec la commande ```ip a```


## Aggrégation des liens 

<details>
  
<summary>Utilité de cette configuration avant déploiement</summary>

- Augmenter la bande passante
Si un serveur effectue des communications avec de gros débits, agréger deux interfaces permet d’additionner leur bande passante (ex: 2 × 1 Gbps = jusqu’à 2 Gbps, théoriquement, en fonction du type de bond et du switch derrière).

- Améliorer la résilience / tolérance aux pannes
Si une carte réseau ou un câble lâche, le lien reste disponible. Avec active-backup ou LACP, une interface peut tomber sans que le serveur soit coupé du réseau. C’est particulièrement important si ce réseau privé est critique (stockage iSCSI, cluster, ou interconnexion entre VM/serveurs).

</details>



### Mise en place :

Pour agréger des liens sous Ubuntu nous allons faire du bonding.

On commence par installer ce paquet ```apt install ifenslave```

Il faut ensuite modifier la configuration netplan pour intégrer ce bonding (Ubuntu 18.04+) :


```
network:
    version: 2
    ethernets:
        public0:
            dhcp4: no
            addresses: [192.168.150.10/24]
        private0:
            dhcp4: no
        private1:
            dhcp4: no
    bonds:
        bond0:
            interfaces: [private0, private1]
            addresses: [192.168.50.10/24]     # IP du bond sur le réseau privé
            parameters:
                mode: active-backup
                primary: private0               # Interface principale (facultatif, mais conseillé)
                mii-monitor-interval: 100
            dhcp4: no
```


<details>
  
<summary>Détails des paramètres de la configuration</summary>

mode: active-backup	    Une interface principale et une secondaire qui prend le relais en cas de panne.
primary: private0	    Définit quelle interface est prioritaire tant qu’elle est active.
mii-monitor-interval	Intervalle de vérification de l’état des liens en millisecondes.

</details>


On applique cette nouvelle configuration avec ```netplan apply```. 


### Vérifications et tests

On peut vérifier notre nouvelle interface virtuelle avec ```ip addr show bond0```. On devrait obtenir quelque chose comme :

```
5: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 2e:79:6f:94:60:52 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.10/24 brd 192.168.50.255 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::2c79:6fff:fe94:6052/64 scope link
       valid_lft forever preferred_lft forever
```

Pour obtenir plus de détails sur ce bond, on peut utiliser cette commande : ```cat /proc/net/bonding/bond0```

```
Ethernet Channel Bonding Driver: v5.15.0-134-generic

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: private0 (primary_reselect always)
Currently Active Slave: private0
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0
Peer Notification Delay (ms): 0

Slave Interface: private1
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 2
Permanent HW addr: 00:0c:29:f5:2d:42
Slave queue ID: 0

Slave Interface: private0
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 1
Permanent HW addr: 00:0c:29:f5:2d:38
Slave queue ID: 0
```


La configuration semble bonne, on peut maintenant simuler une défaillance d'une interface réseau, je vais donc désactiver sur VMware l'interface de mon réseau que j'ai configuré comme étant primaire, pour voir si on passe sur ma seconde interface automatiquement.

Sans le moindre délai (ou du moins, dans un délai très faible, 100ms comme on l'a configuré), en refaisant la commande ```cat /proc/net/bonding/bond0``` je peux voir que ma nouvelle interface "esclave" active est private1 :

```
Bonding Mode: fault-tolerance (active-backup)
Primary Slave: private0 (primary_reselect always)
Currently Active Slave: private1
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0
Peer Notification Delay (ms): 0
```


On peut également noter la modification d'une valeur, celle de la ligne ```Link Failure Count``` sur mon interface récemment désactivée, cette valeur, représentant un décompte de panne détéctées des interfaces réseaux, s'est donc incrémentée :

```
Slave Interface: private0
MII Status: down
Speed: Unknown
Duplex: Unknown
Link Failure Count: 2
Permanent HW addr: 00:0c:29:f5:2d:38
Slave queue ID: 0
```



## QoS avec une limite de bande passante


<details>
  
<summary>Utilité de cette configuration avant déploiement</summary>

Nous allons utiliser tc (traffic control) avec un filtre sur l’IP et le port utilisé pour les sauvegardes pour réaliser cette QoS (quality of service).

L'utilité de cette configuration est de limiter la bande passante pour les sauvegardes afin de ne jamais saturer le réseau pour des sauvegardes qui peuvent être volumineuses et pourraient donc encombrer le réseau, empêchant les autres services de fonctionner correctement. La spécification de l'IP et du port cible permet de cibler efficacement le bon serveur et le bon service afin de limiter uniquement las sauvegardes.

</details>


### Préparation de la configuration

Il faut commencer par identifier le port ou l’IP cible, nous allons simuler, par exemple, des sauvegardes passant par un serveur avec l’IP 192.168.50.20 (donc sur le même réseau que notre bond0) et le port 873 (rsync).

On commence par installer (ou vérifier son installation) iproute2 : ```apt install iproute2```, on peut ensuite nettoyer toute configuration avec tc déjà existante pour être au plus propre avec ```sudo tc qdisc del dev bond0 root || true```.


### Créer la hiérarchie avec un htb

On rentre ces commandes pour créer notre hérarchie :

```
tc qdisc add dev bond0 root handle 1: htb default 20

```


- tc qdisc add dev bond0 root : on ajoute une « discipline de file d'attente » (queuing discipline) sur l’interface bond0.
- handle 1: : on crée un identifiant interne (1:) pour cette discipline.
- htb : on choisit la méthode HTB (Hierarchical Token Bucket), très utilisée pour de la QoS.
- default 20 : si aucun filtre ne correspond à un paquet, il sera automatiquement attribué à la classe d’identifiant 1:20 (on pourra la créer plus tard si besoin).


```
tc class add dev bond0 parent 1: classid 1:1 htb rate 1gbit ceil 1gbit
```


Ici, on créer une classe principale (parent 1:) avec :
- classid 1:1 : identifiant unique pour cette classe principale.
- rate 1gbit : débit garanti (1 gigabit) — tu peux ajuster selon la vitesse réelle de ta carte ou lien.
- ceil 1gbit : plafond maximum autorisé (même valeur ici).


### Configurer la classe limitée à 100 Mbps

```
tc class add dev bond0 parent 1:1 classid 1:10 htb rate 100mbit ceil 100mbit
```


On créer une classe « enfant » sous la classe principale.
- parent 1:1 : cette classe dépend directement de la classe principale.
- classid 1:10 : identifiant unique pour cette classe secondaire.
- rate 100mbit : débit garanti de 100 Mbps.
- ceil 100mbit : plafond de 100 Mbps.


A cette classe, on va ajouter un filtre pour cibler l’IP et le port de destination et rediriger vers cette classe
bash
Copier
Modifier
sudo tc filter add dev bond0 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.50.20/32 flowid 1:10
tc filter add dev bond0 : on ajoute un filtre sur bond0.

protocol ip : ce filtre concerne les paquets IP.

parent 1:0 : on place le filtre au niveau racine (niveau 1:0).

prio 1 : priorité du filtre (1 = priorité haute, il sera vérifié avant les autres).

u32 : type de filtre très flexible qui permet de faire des matchs binaires.

match ip dst 192.168.50.20/32 : ici, on cible tout trafic à destination de l’IP 192.168.50.20.

