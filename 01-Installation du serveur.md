# 01-Installation du serveur

On se concentre ici sur l'installation du serveur utilisé pour le projet AIS phase 1.

1. [Creation de la machine virtuelle](#creation-de-la-machine-virtuelle)
2. [Installation d'Ubuntu](#installation-dubuntu)

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

## Installation d'Ubuntu

Vient ensuite l'installation d'Ubuntu lors du premier lancement de la machine.
