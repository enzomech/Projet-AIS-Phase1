# 01-Installation du serveur

On se concentre ici sur l'installation du serveur utilisé pour le projet AIS phase 1.

1. [Creation de la machine virtuelle](#creation-de-la-machine-virtuelle)
2. [Installation systeme](#installation-systeme)


---


## Creation de la machine virtuelle


J'ai choisi d'utiliser l'hyperviseur VMware® Workstation 17 Pro (17.6.1 build-24319023) pour mon projet.


Il faut d'abord créer une nouvelle machine virtuelle sur VMware, en installation personnalisée

![Capture 1](https://github.com/user-attachments/assets/9f7f771a-abed-44a9-a860-bfda843119c7)


Rajouter l'ISO d'Ubuntu (V22.04.5 live server ici) disponible avec ce lien de téléchargement direct :
https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso


Séléctionner ensuite le bon ISO

![Capture 2](https://github.com/user-attachments/assets/e7efc061-aa82-4bb3-a41f-87f7a8c3f1ad)


J'ai fais le choix d'accorder temporairement une puissance démesurée à la machine pour nos besoins, afin de booster le processus d'installation.
(un choix cohérent pour plus tard serait de donner 2 coeurs de CPU + 4gb de RAM lorsque tous les services seront en fonction)

![Capture 3](https://github.com/user-attachments/assets/3e227212-8fc8-4790-ab6c-0a4ab8ff7348)


![Capture 4](https://github.com/user-attachments/assets/513ef249-7e17-46bc-b15a-2f7923a0b3f4)



Terminer la configuration sur VMware, couper la machine qui va se lancer toute seule, puis aller dans les paramètres de la VM, rajouter les 2 autres disques de 20gb (+ celui déjà rajouté pendant la configuration de la VM).


![Capture 5](https://github.com/user-attachments/assets/29f20f9d-a578-4b69-96a1-04dd7f6305fe)


La configuration finale devrait ressembler à ça :

![Capture 6](https://github.com/user-attachments/assets/87d6994e-7cbd-4427-9794-8364960be596)



La machine est prête à être lancée.


---


## Installation systeme


Vient ensuite l'installation d'Ubuntu avec le RAID 5 et LVM lors du premier lancement de la machine.


<details>
  
<summary>Utilité de cette configuration avant déploiement</summary>

- RAID 5 apporte la tolérance aux pannes disque et la continuité de service (comme pour le réseau qu'on va configurer ensuite, l'idée est la fiabilité).
- LVM permet la flexibilité pour redimensionner ou ajouter de l’espace disque si le serveur doit accueillir plus de bases de données ou de fichiers partagés plus tard.

</details>



### Mise en place :


On suit la configuration classique d'ubuntu, jusqu'à arriver à cette étape où il faut séléctionner "custom storage layout":

![Capture 2](https://github.com/user-attachments/assets/bda2deae-1976-4481-b1f6-7325ac89133d)


Configurer chacun des 3 périphériques pour le démarrage, il est important de faire les mêmes configurations sur les 3 disques, puisqu'elles sont sensées être uniforme pour fonctionner en RAID:

![Capture 3](https://github.com/user-attachments/assets/15e39d5a-60a0-43e4-b8c0-15b623b67988)


On ajoute ensuite une partition GPT pour chacun des 3 périphériques, remplissant l'espace disque disponible à chaque fois, mais il faut les laisser non formatés.

![Capture 4](https://github.com/user-attachments/assets/a14563b6-257a-4716-9344-2e4d2484502f)


On choisit finalement de créer le RAID logiciel (create software RAID), on configure le RAID level sur 5, et on séléctionne toutes les partitions avant de séléctionner créer.

![Capture 5](https://github.com/user-attachments/assets/487b1d52-9f4d-4d77-84ea-51c8f7110921)


Notre RAID est configuré, un tier de l'espace de nos 60go n'est donc plus disponible et est utilisé pour faire fonctionner le RAID 5, on créer maintenant le groupe de volume LVM, on séléctionne le périphérique qui porte le nom de notre RAID et on séléctionne créer.

![Capture 6](https://github.com/user-attachments/assets/840d982d-cf9b-407d-9bdd-233351e93e3d)


On a notre RAID et notre LVM de configuré, il nous reste à créer nos partitions en séléctionnant l'espace libre de notre partition portant le nom de notre LVM, 

![Capture 7](https://github.com/user-attachments/assets/787b263b-e086-4e4a-90a0-95ed51de9ef1)


On répète donc cette étape pour configurer ces 4 partitions :
- web 1G /web
- database 2G /database/
- backup 5G /backup
- root 15G /


Notre configuration finale ressemble à ceci :

![Capture 8](https://github.com/user-attachments/assets/4b023d60-62e8-49e2-8272-3f99b027b1b4)


On poursuit ensuite la configuration d'Ubuntu jusqu'à la fin, puis après le redémarage on peut vérifier notre configuration de stockage avec la commande ```lsblk```

![Capture 9](https://github.com/user-attachments/assets/9207365d-408e-4377-a22a-8c190d78a9cb)


