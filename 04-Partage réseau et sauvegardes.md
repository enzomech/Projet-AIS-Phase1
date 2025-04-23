# 04-Partage réseau et sauvegardes

On se concentre ici sur la configuration du partage réseau avec Samba4 et sur les sauvegardes ainsi que la mise en place de politique de rétentions.

1. [Configuration du partage Samba4](#Configuration-du-partage-Samba4)
2. [Script de sauvegarde](#Script-de-sauvegarde)
3. [Synchronisation automatique](#Synchronisation-automatique)
4. [Rétention des sauvegardes](#Rétention-des-sauvegardes)

---


## Nommer les intefaces réseaux

On commence par mettre à jour notre système

```
sudo apt update && sudo apt upgrade
```


Puis on installe Samba

```
sudo apt install -y samba
```

On créer ensuite notre dossier /shared, on le configure et sécurise correctement, en spécifiant le nobody:nogroup et en donnant toutes les permisssions à tous pour éviter tout problème d'accéssibilité

```
sudo mkdir -p /shared
sudo chown nobody:nogroup /shared
sudo chmod 0777 /shared
```

Et on va ajouter un partage dans ```/etc/samba/smb.conf``` avec cette configuration à la fin du fichier :

```
[shared]
   path = /shared
   browseable = yes
   writable = yes
   guest ok = yes
   read only = no
   create mask = 0777
   directory mask = 0777
```


## Script de sauvegarde




## Synchronisation automatique




## Rétention des sauvegardes




