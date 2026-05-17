+++
title = "Comment j’automatise la sauvegarde de l’ordinateur portable de Madame"
#title = "How do I automatically backup my wife’s laptop with restic"
date = "2026-05-10T18:54:45+02:00"
draft = false
tags = ["restic", "backup", "windows"]
categories = ["system", "network"]
+++

{{< alert >}}
**Attention :** article en cours d’élaboration.
{{< /alert >}}

Ma femme utilise un ordinateur portable sous Windows. Pour elle, l’utilisation 
de cet outil doit se limiter au strict minimum (elle préfère, de très loin, son 
smartphone). Autant dire que mes encouragements à mettre en place une procédure 
de sauvegarde automatique pour éviter de perdre toutes ses données n’ont jamais 
donné lieu qu’à des… "Mouais, je verrai ça un jour." À savoir que pendant 
plusieurs années, la seule sauvegarde qu’elle trainait était une vieille clé 
USB, solution incertaine au possible.

Désespéré, j’ai fini par entreprendre cette tâche par moi-même, en passant par 
des solutions diverses qui ont suivi ma propre courbe d’apprentissage dans le 
domaine. D’abord, sauvegarde occasionnelle sur un disque dur externe. Ensuite, 
j’ai eu l’idée de tenter de gérer la situation via la solution Onedrive de 
Microsoft. On ne m’y reprendra plus. Finalement, je suis passé à un logiciel 
(Perfect Backup) permettant des sauvegardes incrémentales et automatiques. Mais 
un bug sur une sauvegarde m’en a découragé.

J’ai alors décidé d’aller vers des solutions ayant une saveur plus opensource, 
avec l’idée d’avoir quelque chose de solide et (surtout) automatisable pour 
pouvoir (enfin) cesser d’y penser. Je vous présente ici les solutions que j’ai 
construites.

## Stratégie de backup et logiciels

Tout d’abord, on va rappeler une règle basique de la sauvegarde personnelle. 
Idéalement, un système de backup sérieux doit suivre la règle des 3-2-1: pour 
être safe, vos données doivent exister en trois copies, sur au moins deux 
supports différents, dont l’un au moins se trouve hors site. Pour les données de 
ma femme, j’ai opté pour une double sauvegarde incrémentale : une sur mon NAS 
(un Synology DS213j), une sur Google Drive où ma femme dispose d’un espace de 
stockage de 100 Go.

Pour les logiciels j’ai longuement exploré les options disponibles. Exit les 
gros GUI gourmands en ressources. Étant allergique au clavier non-bépo de ma 
femme et par souci d’efficacité, je voulais un outil que je puisse administrer à 
distance depuis mon poste, via une connexion SSH. J’aurais pu utiliser borg, 
outil réputé et fiable, comme je le fais pour la plupart de mes sauvegardes 
personnelles, mais ce n’était pas possible sur Google Drive (ou tout autre 
hébergeur de stockage mainstream que ma femme pourrait utiliser). J’ai alors 
opté pour les solutions suivantes:
- `restic` : un logiciel à la réputation solide. Avec un utilitaire en ligne de 
commande, il se prête facilement à une automatisation via des scripts 
PowerShell. Autres avantages: déduplication, integrity checks, easy (and 
partial) restores, transportable repositories.
- `rclone` : ce petit utilitaire complémentaire servira de backend à restic pour 
pouvoir accéder à Google Drive.
- `OpenSSH` : Pour la couche transport: on fait le choix d’utiliser SSH plutôt 
que SMB, pour plusieurs raisons ; c’est le protocole que j’utilise le plus par 
ailleurs, c’est donc mentalement plus simple. Complement: no persistent network 
mounts, fewer Windows credential problems,  simpler scripting, cleaner failure 
modes, and Linux-like semantics.

### Prerequisites
- Installing `PowerShell 7` on the laptop (link interne)
- Installing `OpenSSH` on the laptop (link interne)
- Installing `micro` editor on the laptop (link interne)
- Set up wife’s SSH access to NAS
- Optional: enable `sshd` on the laptop and set up my user SSH access to the laptop
- Optional: mounting `sshfs` access to my wife’s laptop on my desktop for vim edition

## Installation de restic

Pour commencer nous allons installer restic :
- Vous pouvez trouver la dernière version sur la page « Releases » du dépôt 
github <https://github.com/restic/restic/releases>. Dans la dernière release, 
trouvez la section « Assets » et téléchargez le fichier nommé 
`restic_<version>_windows_amd64.zip`.
- Pour la suite des opérations vous pouvez travailler soit avec un explorateur 
de fichiers soit depuis PowerShell. Dans les deux cas, l’application utilisée 
doit être lancée comme « Administrateur ».
- Créez un dossier ``C:\Tools\restic``. Extrayez le fichier exécutable 
(typiquement: `restic_0.18.1_windows_amd64.exe`) vers ce dossier.
- Renommez ce fichier en `restic.exe`. Cela sera utile pour la phase 
d’automatisation.

## Premier backup

## Automatisation des backups

## Surveillance

# TODO/ADDITIONS
- pour l’installation de `restic`, tester process complet en mode PowerShell 
afin de simplifier le process, à proposer optionnellement :
  - extract: `Expand-Archive -Path .\restic_0.18.0_windows_amd64.zip 
-DestinationPath C:\Tools\restic`
  - rename: `Rename-Item .\restic_0.18.1_windows_amd64.exe restic.exe`
- rclone / google drive ?
