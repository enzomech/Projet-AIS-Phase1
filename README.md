# Projet AIS Phase 1

Ce dépôt contient la documentation et les fichiers associés au projet du titre pro AIS phase 1.

## Objectif

Le projet consiste en l'installation, la configuration et la sécurisation d'un serveur Ubuntu 22.04 avec les composants suivants :
- RAID logiciel (RAID 5) et LVM
- Configuration réseau avancée avec QoS et agrégation de liens
- Serveur LEMP (Nginx, MariaDB, PHP) pour héberger un site web
- Automatisation de tâches avec des scripts shell
- Sauvegardes et synchronisations planifiées
- Sécurisation des accès et des services réseau

---

## Table des matières

1. [Prérequis](#prérequis)
2. [Étapes principales du projet](#etapes-principales-du-projet)
   - [Installation du serveur](#installation-du-serveur)
   - [Configuration réseau](#configuration-reseau)
   - [Mise en place du serveur LEMP](#mise-en-place-du-serveur-lemp)
   - [Partage réseau et sauvegardes](#partage-reseau-et-sauvegardes)
   - [Sécurisation et scripting](#securisation-et-scripting)

---

## Prérequis

### Matériel :
- Trois disques de 20 Go pour le RAID 5 et LVM
- Trois interfaces réseau pour la configuration avancée

### Logiciels :
- Ubuntu Server 22.04
- Samba4 pour les partages réseau
- Outils pour le scripting (bash, cron)
- Outils réseau (tc, wondershaper)

---

## Étapes principales du projet

### Installation du serveur
- Configuration d'un RAID logiciel (RAID 5) et LVM pour gérer les partitions suivantes :
  - `/web` (1 Go)
  - `/database` (2 Go)
  - `/backup` (5 Go)
  - `/` (15 Go)

### Configuration réseau
- Renommer les interfaces réseau :
  - `eth0` → **public0**
  - `eth1` → **private0**
  - `eth2` → **private1**
- Agréger les liens `private0` et `private1`
- Appliquer une QoS avec une limite de 100 Mbps pour les connexions de sauvegarde

### Mise en place du serveur LEMP
- Installation de Nginx, MariaDB et PHP
- Hébergement d'un site web sur le volume `/web`
- Création d'une base de données dédiée avec un compte utilisateur
- Développement d'un script PHP pour tester la connexion entre le serveur web et la base de données

### Partage réseau et sauvegardes
- Partage réseau Samba4 configuré pour `/shared`
- Automatisation des sauvegardes des bases de données et fichiers web vers `/backup`
- Synchronisation des archives de sauvegardes vers `/shared`
- Rétention des sauvegardes :
  - Derniers 2 jours
  - Dernières 2 semaines
  - Derniers 5 mois

### Sécurisation et scripting
- Mise en place d'une routine de mises à jour système
- Sécurisation des accès SSH (authentification par clé, restrictions)
- Firewall : ouverture des ports nécessaires uniquement
- Scripts pour la création automatisée de comptes utilisateur (it, backup, web)
