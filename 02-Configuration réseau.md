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



</details>



### Mise en place :

Pour agréger des liens sous Ubuntu nous allons faire du bonding.

On commence par installer ce paquet ```apt install ifenslave```

Il faut ensuite charger le module bonding avec ```modprobe bonding```, on peut ensuite vérifier avec ```lsmod | grep bonding```

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
            parameters:
                mode: active-backup
                primary: private0
                mii-monitor-interval: 100
            dhcp4: no
            addresses: [192.168.200.10/24]
            nameservers:
                addresses: [8.8.8.8, 1.1.1.1]
```

