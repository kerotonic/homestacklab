+++
title = "Restic : comment j’automatise la sauvegarde du laptop de Madame"
#title = "How do I automatically backup my wife’s laptop with restic"
date = "2026-05-10"
lastmod = "2026-05-31"
draft = false
tags = ["system", "restic", "backup", "windows"]
+++

<!-- TODO/ADDITIONS
- poursuivre rédaction : rclone / google drive ?
- chercher à automatiser mises à jour des outils -->

> [!note] Note
> Article en cours d’élaboration.

> **Logiciels utilisés :**
> - **Client :** Windows 11 Famille, PowerShell 7.6, OpenSSH 9.5p2
> - **Outils :** Restic 0.18.1, Rclone 1.66.0 (TODO : recheck)
> - **Serveur :** Synology DS213j (DSM v7.1.1)

&nbsp;

Ma femme utilise un ordinateur portable sous Windows. Pour elle, l’utilisation 
de cet outil doit se limiter au strict minimum (elle préfère, de très loin, son 
smartphone). Autant dire que mes encouragements à mettre en place une procédure 
de sauvegarde automatique pour éviter de perdre toutes ses données n’ont jamais 
donné lieu qu’à des… « Mouais, je verrai ça un jour. » À savoir que pendant 
plusieurs années, la seule sauvegarde qu’elle traînait était une vieille clé 
USB, solution incertaine au possible.

Désespéré, j’ai fini par entreprendre cette tâche par moi-même, en passant par 
des solutions diverses qui ont suivi ma propre courbe d’apprentissage dans le 
domaine. D’abord, sauvegarde régulière sur un disque dur externe. Ensuite, j’ai 
eu l’idée de tenter de gérer la situation via la solution OneDrive de Microsoft. 
On ne m’y reprendra plus. Finalement, je suis passé à un logiciel (Perfect 
Backup) permettant des sauvegardes incrémentales et automatiques. Mais un bug 
sur une sauvegarde m’en a découragé.

J’ai alors décidé d’aller vers des solutions davantage orientées open source, 
avec l’idée d’avoir quelque chose de solide et (surtout) automatisable pour 
pouvoir (enfin) cesser d’y penser. Je vous présente ici les solutions que j’ai 
construites.


## Stratégie de sauvegarde {#backup-strategy}

Tout d’abord, on va rappeler une règle basique de la sauvegarde personnelle. 
Idéalement, un système de backup sérieux doit suivre la règle du « 3-2-1 » : 
pour être en sécurité, vos données doivent exister au moins en trois copies, sur 
au moins deux supports différents, dont l’un au moins se trouve hors site. Pour 
les données du laptop, j’ai opté pour une double sauvegarde incrémentale : une 
sur notre NAS familial (un Synology DS213j) et une sur Google Drive où ma femme 
— qu’on appellera désormais ici « Julie » — dispose d’un espace de stockage de 
100 Go.

Pour les logiciels, j’ai longuement exploré les options disponibles. Exit les 
gros GUI gourmands en ressources. Étant allergique au clavier non-bépo de Julie 
et par souci d’efficacité, je voulais un outil que je puisse administrer à 
distance depuis mon poste, via une connexion SSH. J’aurais pu utiliser Borg, 
outil réputé et fiable, comme je le fais pour la plupart de mes sauvegardes 
personnelles, mais ce n’était pas possible sur Google Drive (ou tout autre 
hébergeur de stockage mainstream que Julie pourrait utiliser). J’ai alors opté 
pour les solutions suivantes :
- **Restic** : un outil en ligne de commande open source à la réputation solide. 
Il se prête facilement à une automatisation via des scripts PowerShell et 
optimise le stockage grâce à la déduplication de données qui permet des 
sauvegardes incrémentales. Il intègre également des tests de vérification 
d’intégrité et permet des restaurations partielles ou totales.
- **OpenSSH** : ce protocole a été préféré à SMB pour sa parfaite intégration 
dans mon architecture existante. L'utilisation de SSH présente de nombreux 
avantages pour la gestion des sauvegardes : absence de montages réseau 
persistants, suppression des conflits d'identifiants Windows, scripts 
simplifiés, gestion des erreurs plus lisible et comportement natif de type 
Linux.
- **Rclone** : ce couteau suisse du stockage servira de passerelle à Restic pour 
lui permettre d'accéder à Google Drive.


## Prérequis {#system-prerequisites}

Les prérequis suivants sont nécessaires pour mener à bien la procédure :
- **Sur le NAS :**
  - Création d’un compte pour Julie (TODO : lien interne)
  - Activation et configuration du service SSH (TODO : lien interne)
- **Sur le laptop :**
  - Installation de PowerShell 7, OpenSSH et l’éditeur Micro 
(TODO : lien interne)
- **Optionnel :**
  - Activation du démon `sshd` sur le laptop de Julie et configuration d’un 
accès SSH pour administrer la machine à distance depuis son poste fixe
  - Mise en place d’un montage `sshfs` pour éditer les fichiers du laptop 
directement depuis le poste de travail avec son éditeur favori (Vim, par 
exemple)


## Installation de Restic {#restic-installation-windows}

Pour installer Restic sur Windows, on trouve la dernière version sur la page « 
Releases » du [dépôt GitHub](https://github.com/restic/restic/releases). Dans la 
dernière release, il faut repérer la section « Assets » et télécharger le 
fichier nommé `restic_<version>_windows_amd64.zip`.

> [!important] Important
> Pour la suite du document, il faut adapter les noms d’utilisateur, chemins et 
versions à votre configuration.

Une fois Restic téléchargé, voilà la marche à suivre :
- On ouvre un terminal PowerShell (ou un explorateur de fichiers si on préfère) 
pour faire les manipulations nécessaires.
- On crée un dossier `C:\Tools\restic` et on s’y rend :
```powershell
mkdir C:\Tools\restic
cd C:\Tools\restic
```
- On déplace l’archive téléchargée vers ce dossier puis on extrait le fichier 
exécutable de l’archive :
```powershell
Move-Item -Path "C:\Users\julie\Downloads\restic_0.18.1_windows_amd64.zip" `
  -Destination .
Expand-Archive -Path ".\restic_0.18.1_windows_amd64.zip" `
  -DestinationPath .
```
- On renomme ce fichier en `restic.exe`, ce qui sera utile pour la phase 
d’automatisation :
```powershell
Rename-Item -Path ".\restic_0.18.1_windows_amd64.exe" -NewName "restic.exe"
```


## Accès SSH au NAS {#nas-ssh-access}

Restic communiquera avec le NAS en passant par une connexion SSH, il faut donc 
que l’utilisateur Julie ait un accès autorisé. Par ailleurs, il faut que cet 
accès puisse se faire via une [clé SSH](/toolbox/glossary#ssh-key), ce qui est 
obligatoire pour automatiser la sauvegarde.


### Création d’une clé SSH {#powershell-ssh-key}

Pour créer une clé SSH sur le laptop, on ouvre un terminal PowerShell depuis la 
session de Julie, on s’assure qu’on est dans le bon dossier (`C:\Users\julie`) 
et on génère la clé :
```powershell
cd $HOME
ssh-keygen -t ed25519 `
  -f .ssh\id_ed25519 `
  -N "" `
  -C "${env:USERNAME}@${env:COMPUTERNAME}_$(Get-Date -Format yyyy-MM-dd)"
```

On vérifie ensuite que les fichiers ont bien été créés :
```powershell
ls .ssh
```

Dans la liste affichée, devraient figurer deux fichiers comme ceux-là :
```
-a---          22/05/2026    10:23            419 id_ed25519
-a---          22/05/2026    10:23            112 id_ed25519.pub
```

Ces deux fichiers fonctionnent ensemble :
- `id_ed25519` est la clé privée, qui ne doit jamais sortir de ce dossier.
- `id_ed25519.pub` est la clé publique, que l’on installera sur le NAS.

On peut visualiser la clé publique à l’aide d’une simple commande :
```powershell
cat .ssh\id_ed25519.pub
```

Elle doit ressembler à :
```
ssh-ed25519 AAAARNNzaC1lZDI1OTE5AAAAIPGDQJzIiLkq69g6lb+gpsaWL6VOHtqQbUYaePuTDCJG Julie@laptop_2026-05-22
```


### Installation de la clé publique sur le NAS {#ssh-key-synology}

On part ici du principe que Julie a un compte sans droits administrateur sur le 
NAS où on veut installer la clé. Si elle avait un compte administrateur, la 
procédure serait un peu plus simple, mais on préfère ici envisager le cas le 
plus difficile (et le plus sécurisé). Comme on dit, qui peut le plus peut le 
moins.

Normalement, si on a installé proprement notre NAS et qu’on a déjà activé le 
[service SSH](/toolbox/glossary#ssh-server), on dispose d’un autre utilisateur 
qui a déjà un accès SSH opérationnel et qui a les droits d’administration (on va 
l’appeler « nico »). C’est lui qui va bosser pour ouvrir l’accès à Julie.

> [!info]- Pour les curieux : suis-je vraiment obligé de passer par un autre utilisateur ? C’est relou…
> Techniquement non. Mais :
> - Je pars du principe que le démon `sshd` d’un serveur NAS configuré de 
manière sécurisée interdit, de préférence, la connexion par mot de passe 
(`PasswordAuthentication no` dans `/etc/ssh/sshd_config`). Dans ce cas, il est 
inévitable d’utiliser un autre utilisateur ayant déjà un accès, sauf à réactiver 
temporairement l’authentification par mot de passe.
> - En plus de cela, par défaut, sur DSM 7.1.1, un utilisateur sans droits 
d’administration n’a pas accès à un [shell](/toolbox/glossary#shell), ce qui lui 
interdit d’ouvrir une session SSH interactive. C’est pourquoi de nombreux 
tutoriels sur Synology indiquent d’attribuer les droits d’administration à 
l’utilisateur concerné. Personnellement, je ne souhaite pas donner ces droits à 
tous les membres de la famille. Une autre solution est alors de modifier 
manuellement le fichier `/etc/passwd` et d’attribuer un shell à Julie. Il faut 
donc obligatoirement qu’un autre utilisateur (administrateur) se connecte pour 
modifier ce fichier au préalable.
> - Bien sûr, si Julie a un compte administrateur et si le serveur SSH autorise 
la connexion avec mot de passe, elle peut faire elle-même les manipulations 
nécessaires. Elle n’aura alors pas besoin de toucher au fichier `/etc/passwd`. 
Il faut toutefois savoir qu’elle ne pourra pas utiliser `ssh-copy-id`, qui est 
l’utilitaire normalement prévu pour copier une clé SSH, parce que la version 
d’OpenSSH utilisée par défaut sur Windows n’intègre pas cet outil. Il lui faudra 
se connecter par mot de passe et créer le fichier `authorized_keys` (voir 
ci-dessous).

Pour copier la clé sur le NAS, on suit les étapes suivantes :
- On se connecte au serveur (ici : `nas.lan`) :
```bash
$ ssh nico@nas.lan
```
- On crée un dossier `.ssh` dans le répertoire personnel de Julie, en 
considérant que sur le NAS, les dossiers `home` se trouvent dans 
`/var/services/homes/<user>` :
```bash
$ cd /var/services/homes/Julie/
$ sudo mkdir -p .ssh
```
- Avec [Vim](/toolbox/vim-survival-guide), on crée (ou ouvre, s’il existe déjà) 
le fichier `authorized_keys` :
```bash
$ sudo vim .ssh/authorized_keys
```
- On colle dans le fichier la clé que l’on avait affichée à la fin de la section 
[Création d’une clé](#powershell-ssh-key). On sauve, on sort.
- Il reste alors à ajuster le propriétaire (avec `chown`) et les permissions 
(avec `chmod`) du dossier et du fichier (pour des explications sur la 
signification de ces commandes, voir ici (TODO : lien interne)) :
```bash
$ sudo chown -R Julie:users .ssh
$ sudo chmod 700 .ssh
$ sudo chmod 600 .ssh/authorized_keys
```


### Modification de `/etc/passwd` {#etc-passwd-editing}

Par défaut, sur certains NAS Synology (mon DS213j, en tout cas), le fichier 
`/etc/passwd` interdit à un utilisateur non administrateur d’avoir accès à un 
[shell](/toolbox/glossary#shell). Il faut donc modifier ce fichier.

> [!warning] Attention
> Avant de modifier ce fichier, il vaut mieux en faire une copie 
de sécurité comme indiqué.

```bash
$ sudo cp -a /etc/passwd /etc/passwd.bkp
$ sudo vim /etc/passwd
```

Une fois ce fichier ouvert, on modifie la ligne correspondant à Julie. La ligne 
ciblée doit ressembler à ceci :
```
Julie:x:1027:100::/var/services/homes/Julie:/usr/bin/nologin
```

L’ensemble du champ après le dernier deux-point (`:`) correspond au shell 
attribué à Julie. Il faut modifier cette ligne (uniquement !), en remplaçant 
`/usr/bin/nologin` (ou `/sbin/nologin`, ou `/bin/false`…) par `/bin/sh`, pour 
obtenir le résultat suivant :
```
Julie:x:1027:100::/var/services/homes/Julie:/bin/sh
```

Il faut faire bien attention à ne rien modifier d’autre dans le fichier. On 
sauve, on sort.


### Test de l’accès SSH {#ssh-test}

Une fois toutes les étapes terminées, on teste l’accès depuis le laptop de 
Julie, via PowerShell :
```powershell
ssh Julie@nas.lan
```

Si ça passe, on est bon. On peut initialiser le dépôt restic !


## Initialisation du dépôt {#restic-repo-init}

Il nous faut maintenant initialiser un dépôt Restic sur le NAS afin d’y stocker 
nos sauvegardes.

1) Depuis notre terminal PowerShell, on commence par créer le dossier qui 
servira pour le dépôt :
```powershell
ssh Julie@nas.lan 'mkdir -p ~/restic-backup'
```

2) Pour pouvoir agir sur un dépôt, la commande `restic` a besoin de deux choses 
: l’adresse du dépôt et un mot de passe. Plutôt que fournir ces deux 
informations à chaque utilisation de `restic`, on peut définir deux variables 
d’environnement : RESTIC_REPOSITORY pour le dépôt, RESTIC_PASSWORD pour le mot 
de passe. Le plus pratique est de définir ces deux variables dans un script 
PowerShell qu’il nous suffira d’exécuter une seule fois au début d’une session 
pour pouvoir utiliser ensuite `restic` sans nous en soucier. Ce script servira 
ensuite aussi lors de la phase d’automatisation. On crée donc un script :
```powershell
cd $HOME
micro .restic_env.ps1
```

3) Dans ce fichier, on copie le code suivant, en veillant à renseigner correctement 
l’identifiant et l’adresse du serveur, et en choisissant un mot de passe.
```powershell
$env:RESTIC_REPOSITORY="sftp:Julie@nas.lan:/home/restic-backup"
$env:RESTIC_PASSWORD="mot-de-passe-au-choix"
```

> [!info]- Pour les curieux : pourquoi `/home/restic-backup` au lieu de `/home/Julie/restic-backup` ?
> Restic accède au dépôt via le protocole SFTP. Sur DSM 7.1.1, le serveur SFTP 
expose le répertoire personnel de l'utilisateur sous la forme d'un dossier 
virtuel `/home`. Le chemin `/home/restic-backup` correspond donc au dossier 
`restic-backup` situé dans le répertoire personnel de Julie.

> [!important] Important
> Le mot de passe du dépôt Restic est indispensable pour accéder aux sauvegardes.
> Si vous le perdez, les données du dépôt deviendront irrécupérables.

4) On peut maintenant initialiser le dépôt, en chargeant d’abord notre script :
```powershell
. $HOME\.restic_env.ps1
restic init
```


## Première sauvegarde {#restic-first-backup}

<!--Il est temps maintenant d’effectuer une première sauvegarde des données.
```
restic backup C:\Users\Julie
```

Sanity check
```
restic -r sftp:Emilie@nas:/home/Emilie/restic-backup snapshots
```
-->

## Automatisation des backups

## Surveillance
