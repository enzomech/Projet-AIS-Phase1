# 05-Sécurisation et scripting

On se concentre ici sur la configuration du partage réseau avec Samba4 et sur les sauvegardes ainsi que la mise en place de politique de rétentions.

1. [Routine de maj système](#Routine-de-maj-système)
2. [Sécurisation SSH](#Sécurisation-SSH)
3. [Configuration du pare-feu](#Configuration-du-pare-feu)
4. [Scripts de création de comptes utilisateurs](#Scripts-de-création-de-comptes-utilisateurs)

---


## Routine de maj système

On commence par créer un script de maj pour le système dans ```/usr/local/bin/update_system.sh``` avec le code suivant :

```
#!/bin/bash

echo "=== $(date) ===" >> /var/log/update_system.log
apt update >> /var/log/update_system.log 2>&1
apt upgrade -y >> /var/log/update_system.log 2>&1
echo "" >> /var/log/update_system.log
```

On lui rajoute les droits d'exécution

```
sudo chmod +x /usr/local/bin/update_system.sh
```

Et pour automatiser l'exécution de ce script, on utilise à nouveau une planification avec Cron, demandant de lancer le script tous les lundi à 02h30, on y rajoute donc une entrée avec ```sudo crontab -e```

Puis on y rajoute cette ligne : ```30 2 * * 1 /usr/local/bin/update_system.sh```

## Sécurisation SSH

On va utiliser une clé pour sécuriser la connection en SSH, on commence donc par générer une clé ssh depuis la machine cliente : ```ssh-keygen```

Puis on copie la clé publique SSH avec cette commande comme si on se connectait en SSH depuis son client vers le serveur

```ssh-copy-id user@ip_du_serveur```

Ce qui donne pour moi ```ssh-copy-id ubzo@192.168.150.10```

On va ensuite modifier les lignes suivantes dans le fichier de config ```sudo nano /etc/ssh/sshd_config``` :

```
PasswordAuthentication no
PermitRootLogin no
```

On finit par redémarrer notre service ssh pour recharger la nouvelle configuration :

```
sudo systemctl restart ssh
```


## Configuration du pare-feu

Avant d'activer le pare-feu, on va le configurer en ouvrant certains ports, notamment celui pour SSH afin d'éviter les coupures intempestives car je travaille en SSH.
```sudo ufw allow OpenSSH```

Il est également utile d'ouvrir le port du HTTP pour pouvoir accéder à notre site sur le server Lamp, (on aurait utilisé le port 443 si on avait sécurisé notre connection en HTTPS)
```sudo ufw allow 80```

On rajoute également le port utilisé par Samba pour notre configuration de sauvegardes
```sudo ufw allow 445```

Et on peut finalement activer le pare feu avec les nouvelles exceptions fraîchement créées
```sudo ufw enable```

On peut vérifier les règles du pare feu à tout moment avec la commande suivante
```sudo ufw status numbered```


## Scripts de création de comptes utilisateurs

On créer le script suivant dans le fichier ```/usr/local/bin/create_users.sh``` :

```
#!/bin/bash

for user in it web backup; do
    if ! id "$user" &>/dev/null; then
        useradd -m -s /bin/bash "$user"
        echo "$user:$(openssl rand -base64 12)" | chpasswd
        echo "Utilisateur $user créé."
    else
        echo "Utilisateur $user déjà existant."
    fi
done
```

Et on rajoute finalement les droits d'exécution à ce script

```sudo chmod +x /usr/local/bin/create_users.sh```


L'infrastructure est finalement légèrement plus sécurisée et possède de quelques outils d'automatisation grace aux scripts, le projet est terminé.
