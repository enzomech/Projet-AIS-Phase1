# 02-Configuration réseau

On se concentre ici sur la configuration du réseau sur notre serveur ubuntu fraîchement installé.

1. [Nommer les intefaces réseaux](#nommer-les-intefaces-réseaux)
2. [Aggrégation des liens](#aggrégation-des-liens)
3. [QoS avec une limite de bande passante](#QoS-avec-une-limite-de-bande-passante)


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


A cette classe, on va ajouter un filtre pour cibler l’IP et le port de destination et rediriger ces paquets vers la classe limitée fraichement créée.

On utilise donc `tc` avec `u32` pour cibler le port rsync (873).

Element important à prendre en compte : le port TCP/UDP est contenu dans les en-têtes du paquet, il faut préciser dans le filtre l’offset correct. * L’offset 20 (0x14) correspond à la position où se trouve le port dans l’en-tête IP+TCP.


<details>
  
<summary>Explication détaillée de cette partie</summary>

(explications détaillées par chatGPT)

---

#### **Contexte : pourquoi parle-t-on d’offset et d’en-tête ?**  
Quand un paquet IP circule sur le réseau, il est composé de plusieurs couches :  

1. **En-tête IP** : contient l’adresse source, l’adresse de destination, le protocole (TCP, UDP…).  
2. **En-tête TCP (ou UDP)** : contient notamment les ports source et destination.  
3. **Les données** : c’est le contenu transporté.  

Le port de destination est dans **l’en-tête TCP**, qui arrive **juste après l’en-tête IP**.  

---

####  **Pourquoi faut-il parler d’offset ?**  
Quand tu crées un filtre `tc u32`, il fonctionne au niveau des paquets bruts.  
Il lit directement les octets du paquet.  
Donc, pour cibler un champ particulier (comme un port), il faut connaître :  
- Où commence ce champ dans le paquet (l’offset)  
- Sa taille (ici, 2 octets pour un port TCP ou UDP)  

---

####  **Structure simplifiée d’un paquet IPv4 + TCP**  
| Partie                   | Taille (octets) |
|--------------------------|-----------------|
| En-tête IP               | En général 20   |
| En-tête TCP              | En général 20   |
| Données                  | variable        |

>  Attention : l’en-tête IP peut avoir une taille variable si des options sont présentes, mais dans **99% des cas sans options**, il fait 20 octets.

---

####  **Position du port TCP de destination**  
- L’en-tête IP (20 octets) est suivi de l’en-tête TCP.  
- L’en-tête TCP commence donc **à l’offset 20 (0x14)** dans le paquet.  
- Le premier champ de l’en-tête TCP est :  
   - Port source (2 octets)  
   - Juste après : port destination (2 octets)  

Donc :  
- Port source = octets 20-21 (offset 20 à 21)  
- **Port destination = octets 22-23 (offset 22 à 23)**  

---

####  **Mais dans la commande `tc filter u32`, comment ça marche ?**  
Quand on écrit :  
```bash
tc filter add dev bond0 protocol ip parent 1:0 prio 1 u32 \
match ip protocol 6 0xff \
match ip dport 873 0xffff \
flowid 1:10
```
- **`match ip protocol 6 0xff`** : on filtre uniquement les paquets TCP (le protocole TCP est identifié par le numéro 6 dans l’en-tête IP).  
- **`match ip dport 873 0xffff`** : c’est une commande simplifiée fournie par `tc` pour ne pas te faire calculer les offsets à la main — elle indique "je veux cibler le port de destination TCP/UDP 873".  
- En réalité, `match ip dport` est un raccourci qui automatise ce calcul et va directement chercher les octets 22-23 dans le paquet.  

---

####  Si tu voulais tout faire "à la main" :  
Tu pourrais écrire un filtre brut `u32` avec offset, par exemple :  
```bash
tc filter add dev bond0 protocol ip parent 1:0 prio 1 u32 \
match ip protocol 6 0xff \
match u16 0x0369 0xffff at 22 \
flowid 1:10
```
- **`match u16 0x0369 0xffff at 22`** : ça veut dire :  
  - Cherche la valeur hexadécimale 0x0369 (873 en hexadécimal) sur 2 octets (`u16`) à partir du 22e octet du paquet.  

Mais heureusement, la syntaxe `match ip dport` simplifie ça en se basant sur les standards des paquets IP + TCP/UDP sans avoir à faire le calcul toi-même.  

---

####  En résumé :
| Ce que tu veux filtrer        | Où ça se trouve dans le paquet           | Comment tc le comprend                   |
|--------------------------------|------------------------------------------|------------------------------------------|
| Le port TCP de destination     | Dans les octets 22-23 (après en-tête IP) | Grâce à `match ip dport` ou avec un offset `at 22` |

---

</details>


```
tc filter add dev bond0 protocol ip parent 1:0 prio 1 u32 \
match ip protocol 6 0xff \
match ip dport 873 0xffff \
flowid 1:10
```

- protocol ip : pour paquets IP.
- prio 1 : priorité élevée.
- match ip protocol 6 0xff : on cible le protocole TCP (6 dans la table des protocoles IP).
- match ip dport 873 0xffff : on cible le port de destination 873.
- flowid 1:10 : on dirige ces paquets vers la classe limitée à 100 Mbps.


### Vérification de la configuration


On peut lancer cette série de tests afin de s'assurer de la conformité de notre configuration :

---

La commande qui affiche les files d’attente actives avec leurs statistiques (packets, drops) :

```
tc -s qdisc show dev bond0
```

<details>
  
<summary>Résultats sur ma machine :</summary>

```
qdisc htb 1: root refcnt 17 r2q 10 default 0x20 direct_packets_stat 2 direct_qlen 1000
 Sent 140 bytes 2 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0


```

</details>

---

La commande qui affiche les classes HTB configurées et leur utilisation (volume, débit) :

```
tc -s class show dev bond0
```

<details>
  
<summary>Résultats sur ma machine :</summary>

```
class htb 1:10 parent 1:1 prio 0 rate 100Mbit ceil 100Mbit burst 1600b cburst 1600b
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
 lended: 0 borrowed: 0 giants: 0
 tokens: 2000 ctokens: 2000

class htb 1:1 root rate 1Gbit ceil 1Gbit burst 1375b cburst 1375b
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
 lended: 0 borrowed: 0 giants: 0
 tokens: 187 ctokens: 187
```

</details>

---

La commande qui liste les filtres appliqués pour diriger le trafic vers les classes :

```
tc filter show dev bond0
```


<details>
  
<summary>Résultats sur ma machine :</summary>

```
filter parent 1: protocol ip pref 1 u32 chain 0
filter parent 1: protocol ip pref 1 u32 chain 0 fh 800: ht divisor 1
filter parent 1: protocol ip pref 1 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 flowid 1:10 not_in_hw
  match 00060000/00ff0000 at 8
  match 00000369/0000ffff at 20
```

</details>

---


### Persistance de la configuration


On remarquera que la configuration tc n’est pas persistante après un redémarrage. Mais après avoir testé et validé la configuration manuelle de tc, on peut en faire un script, qui tournera grace à un service au démarrage de la machine, cela s'adapte bien à notre configuration, et on peut maîtriser les paramètres de celle-ci.

Voici le script que j'ai décidé de créer reprenant ma configuration :  ```/etc/tc-qos-bond0.sh```

```
#!/bin/bash

# Nettoyage
tc qdisc del dev bond0 root || true

# Création HTB
tc qdisc add dev bond0 root handle 1: htb default 20
tc class add dev bond0 parent 1: classid 1:1 htb rate 1gbit ceil 1gbit
tc class add dev bond0 parent 1:1 classid 1:10 htb rate 100mbit ceil 100mbit

# Filtre vers port 873 (TCP)
tc filter add dev bond0 protocol ip parent 1:0 prio 1 u32 \
match ip protocol 6 0xff \
match ip dport 873 0xffff \
flowid 1:10
```

Il ne faut pas oublier d'ajouter les droits d'éxécution pour le script :

```sudo chmod +x /etc/tc-qos-bond0.sh```


Puis on créer un service dans systemd qui fait tourner ce script à chaque lancement après avoir initialisé le network :  ```/etc/systemd/system/tc-qos-bond0.service```

```
[Unit]
Description=TC QoS rules for bond0
After=network.target

[Service]
Type=oneshot
ExecStart=/etc/tc-qos-bond0.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target

```


On active et vérifie notre nouveau service :

```
sudo systemctl daemon-reexec
sudo systemctl enable tc-qos-bond0.service
sudo systemctl start tc-qos-bond0.service
sudo systemctl status tc-qos-bond0.service
```

Puis comme dernier test, on redémarre la machine et affiche notre bond0 :

```
sudo tc -s qdisc show dev bond0
```

<details>
  
<summary>Résultats sur ma machine :</summary>

```
qdisc htb 1: root refcnt 17 r2q 10 default 0x20 direct_packets_stat 2 direct_qlen 1000
 Sent 140 bytes 2 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
```

</details>

### Debuggage

J'ai remarqué en tentant un update de mon système que je n'avais plus accès à internet car la gateway par défaut n'était plus spécifiée et que je n'avais plus de serveur DNS enregistré non plus, j'ai donc remis cela en place :

```
sudo ip route add default via 192.168.150.2
```

Après avoir renseignée ma gateway sur mon NAT VMware, je modifie également le fichier ```/etc/resolv.conf``` spécifiant le DNS (en l'occurence de google) ```nameserver 8.8.8.8```

On test finalement la gateway :

```
ip route
default via 192.168.150.2 dev public0
```

Et on ```ping google.com``` pour vérifier l'accès à internet et le DNS.
