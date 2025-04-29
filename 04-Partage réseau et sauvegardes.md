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

On redémarre le service Samba

```
sudo systemctl restart smbd
```

Et on peut tester en rentrant simplement ```\\192.168.150.10\shared``` dans l'explorateur des fichiers windows de notre hôte.



## Script de sauvegarde

On commence par créer le répertoire de backup et on y accorde les propriétaires et droits appropriés.

```
sudo mkdir -p /backup
sudo chown root:root /backup
sudo chmod 700 /backup
```


Ensuite on créer le script ```/usr/local/bin/backup.sh``` avec le texte suivant :

```
#!/bin/bash

DATE=$(date +'%Y-%m-%d_%H-%M')
DEST="/backup/backup_$DATE"
mkdir -p "$DEST"

# Sauvegarde des fichiers web
tar czf "$DEST/web.tar.gz" /web

# Sauvegarde des bases MariaDB
mysqldump -u root --all-databases > "$DEST/db.sql"

# Nettoyage des permissions
chmod -R 600 "$DEST"/*
```


## Synchronisation automatique

Pour la synchronisation automatique, on va simplement ajouter une ligne de commande utilisant rsync à la fin du script de sauvegarde, pointant vers le dossier partagé, dans un nouveau dossier renseignant la dernière backup.

```
rsync -a --delete "$DEST/" /shared/latest_backup/
mkdir -p /shared/latest_backup
```

## Rétention des sauvegardes

Les sauvegardes seront classées dans ces répertoires pour répondre à une politique de sauvegarde adaptée :

  - Derniers 2 jours ```/backup/daily```
  - Dernières 2 semaines ```/backup/weekly```
  - Derniers 5 mois ```/backup/monthly```

On créer donc les répertoires :

```sudo mkdir -p /backup/daily /backup/weekly /backup/monthly```

Un script de rotation supprimera ensuite les fichiers trop anciens selon des critères précis, le voici ```/usr/local/bin/rotate_backups.sh``` :

```
#!/bin/bash

# Répertoire de base des sauvegardes
BASE_DIR="/backup"

# Jours de conservation
DAILY_KEEP=2
WEEKLY_KEEP=2
MONTHLY_KEEP=5

# Supprimer les sauvegardes trop anciennes
find "$BASE_DIR/daily" -type f -mtime +$((DAILY_KEEP - 1)) -delete
find "$BASE_DIR/weekly" -type f -mtime +$((7 * WEEKLY_KEEP - 1)) -delete
find "$BASE_DIR/monthly" -type f -mtime +$((30 * MONTHLY_KEEP - 1)) -delete
```

On donne ensuite les droits d'exécution pour ce script :

```sudo chmod +x /usr/local/bin/rotate_backups.sh```

Et pour automatiser l'exécution de ce script, on utilise une planification avec Cron, demandant de lancer le script tous les jours à 03h00, on y rajoute donc une entrée avec ```sudo crontab -e```

Puis on y rajoute cette ligne : ```0 3 * * * /usr/local/bin/rotate_backups.sh```

Les sauvegardes sont finalement configurées comme il faut.
