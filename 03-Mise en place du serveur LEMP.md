# 03-Mise en place du serveur LEMP

On passe à l'installation des services nécéssaires et à la mise en place du serveur LEMP.

1. [Installation de Nginx, MariaDB et PHP](#Installation-de-Nginx,-MariaDB-et-PHP)
2. [Hébergement du site web sur le volume web](#Hébergement-du-site-web-sur-le-volume-web)
3. [Création de base de données dédiée](#Création-de-base-de-données-dédiée)


---


## Installation de Nginx, MariaDB et PHP

On commence par mettre à jour notre système

```
sudo apt update && sudo apt upgrade
```


Puis on installe les services nécéssaires à la mise en place du serveur LEMP, à savoir Nginx, MariaDB et PHP.

```
sudo apt install -y nginx mariadb-server php-fpm php-mysql
```


## Hébergement du site web sur le volume web

## Création de base de données dédiée
