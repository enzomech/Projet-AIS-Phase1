# 03-Mise en place du serveur LEMP

On passe à l'installation des services nécéssaires et à la mise en place du serveur LEMP.

1. [Installation de Nginx, MariaDB et PHP](#Installation-de-Nginx,-MariaDB-et-PHP)
2. [Hébergement du site web sur le volume web](#Hébergement-du-site-web-sur-le-volume-web)
3. [Création de base de données dédiée](#Création-de-base-de-données-dédiée)
4. [Script PHP de test de connexion](#Script-PHP-de-test-de-connexion)

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

On prépare le dossier pour un site web fictif "Safeguard" et le sécurise en spécifiant le propriétaire et des droits limités.

```
sudo mkdir -p /web/safeguard
sudo chown -R www-data:www-data /web/safeguard
sudo chmod -R 755 /web/safeguard
```

On rajoute ensuite un fichier de test pour php
```
echo "<?php phpinfo(); ?>" | sudo tee /web/safeguard/index.php
```

Puis on configure le Virtual Host Nginx en créant le fichier ```/etc/nginx/sites-available/safeguard``` et en y rajoutant :

```
server {
    listen 80;
    server_name localhost;

    root /web/safeguard;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Finalememnt, on active le site en créant un lien symbolique de notre site dans les sites activés, on vérifie la syntaxe avec nginx et on le recharge.

```
sudo ln -s /etc/nginx/sites-available/safeguard /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Puis sur l'hôte, on peut tester le site web avec l'adresse ip suivis de notre page de test ```http://192.168.150.10/index.html```, même si une page avec erreur 404 s'affiche, on peut voir le service nginx chargé à travers notre site "nginx/1.18.0 (Ubuntu)"

Le problème vient du fait que j'ai oublié de désactiver le site par défaut avec la commande :

```
sudo rm /etc/nginx/sites-enabled/default
```


## Création de base de données dédiée

On entamme notre connexion à MariaDB :

```
sudo mysql
```


On rentre ensuite les requêtes SQL suivantes :

```
CREATE DATABASE safeguard_db;
CREATE USER 'safeguard_user'@'localhost' IDENTIFIED BY 'motDePasseFort';
GRANT ALL PRIVILEGES ON safeguard_db.* TO 'safeguard_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```


## Script PHP de test de connexion

Pour tester la connexion à la base de donnée, on va créer un fichier dbtest.php dans /web/safeguard en y rajoutant le script suivant :

```
<?php
$host = 'localhost';
$db   = 'safeguard_db';
$user = 'safeguard_user';
$pass = 'motDePasseFort';

try {
    $pdo = new PDO("mysql:host=$host;dbname=$db", $user, $pass);
    echo "Connexion réussie à la base de données.";
} catch (PDOException $e) {
    echo "Erreur : " . $e->getMessage();
}
?>
```

On va ensuite vérifier notre script php à l'adresse ```http://192.168.150.10/dbtest.php``` afin d'y voir apparaître ```Connexion réussie à la base de données.```, notre serveur est opérationnel.
