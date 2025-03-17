# 02-Configuration réseau

On se concentre ici sur la configuration du réseau sur notre serveur ubuntu fraîchement installé.

1. [Nommer les intefaces réseaux](#nommer-les-intefaces-réseaux)
2. [Aggrégation des liens](#aggrégation des liens)


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
            dhcp4: yes
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


