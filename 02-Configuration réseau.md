# 02-Configuration réseau

On se concentre ici sur la configuration du réseau sur notre serveur ubuntu fraîchement installé.

1. [Nommer les intefaces réseaux](#nommer-les-intefaces-réseaux)
2. [Aggrégation des liens](#aggrégation-des-liens)


---


## Nommer les intefaces réseaux

Nommer les interfaces réseaux est très simple sur les versions récentes d'Ubuntu (Ubuntu 18.04+), il suffit de spécifier le nommage dans la configuration réseau de netplan :

```
network:
    version: 2
    ethernets:
        ens33:
            match:
                macaddress: 00:0c:29:f5:2d:2e
            set-name: public0
            dhcp4: no
            addresses:
                - "192.168.150.10/24"
        ens37:
            match:
                macaddress: 00:0c:29:f5:2d:38
            set-name: private0
            dhcp4: yes
        ens38:
            match:
                macaddress: 00:0c:29:f5:2d:42
            set-name: private1
            dhcp4: yes
```

Il est important de spécifier la bonne interface et la bonne adresse MAC, on peut obtenir ces informations avec la commande ```ip link show```.

Pour les anciennes versions d'Ubuntu on peut aussi par exemple utiliser une règle udev.

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
        ens33:
            match:
                macaddress: 00:0c:29:f5:2d:2e
            set-name: public0
            dhcp4: no
            addresses: [192.168.150.10/24]
        ens37:
            match:
                macaddress: 00:0c:29:f5:2d:38
            set-name: private0
            dhcp4: no
        ens38:
            match:
                macaddress: 00:0c:29:f5:2d:42
            set-name: private1
            dhcp4: no
    bonds:
        bond0:
            interfaces: [ens37, ens38]
            parameters:
                mode: active-backup
                primary: ens37
                mii-monitor-interval: 100
            dhcp4: no
            addresses: [192.168.200.10/24]
            nameservers:
                addresses: [8.8.8.8, 1.1.1.1]
```

