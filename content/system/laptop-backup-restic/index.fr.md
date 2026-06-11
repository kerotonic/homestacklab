+++
title = "Restic : comment j’automatise la sauvegarde du laptop de Madame"
#title = "How do I automatically backup my wife’s laptop with restic"
date = "2026-05-10"
lastmod = "2026-06-11"
draft = false
tags = ["system", "restic", "backup", "windows"]
+++

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
dans mon architecture existante. L’utilisation de SSH présente de nombreux 
avantages pour la gestion des sauvegardes : absence de montages réseau 
persistants, suppression des conflits d’identifiants Windows, scripts 
simplifiés, gestion des erreurs plus lisible et comportement natif de type 
Linux.
- **Rclone** : ce couteau suisse du stockage servira de passerelle à Restic pour 
lui permettre d’accéder à Google Drive.


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

Pour installer Restic sur Windows, le plus simple est d’utiliser l’utilitaire de 
gestion de paquets de Windows, WinGet. On ouvre une instance de PowerShell en 
mode administrateur puis on installe Restic :
```powershell
winget install restic.restic --scope machine
```

On vérifie ensuite que l’exécutable est bien installé et accessible :
```powershell
restic version
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
```text
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
mkdir .restic
micro .restic\restic_env.ps1
```

3) Dans ce fichier, on copie le code suivant, en veillant à renseigner 
correctement l’identifiant et l’adresse du serveur, et en choisissant un mot de 
passe.
```powershell
$env:RESTIC_REPOSITORY="sftp:Julie@nas.lan:/home/restic-backup"
$env:RESTIC_PASSWORD="mot-de-passe-au-choix"
```

> [!info]- Pour les curieux : pourquoi `/home/restic-backup` au lieu de `/home/Julie/restic-backup` ?
> Restic accède au dépôt via le protocole SFTP. Sur DSM 7.1.1, le serveur SFTP 
expose le répertoire personnel de l’utilisateur sous la forme d’un dossier 
virtuel `/home`. Le chemin `/home/restic-backup` correspond donc au dossier 
`restic-backup` situé dans le répertoire personnel de Julie.

> [!important] Important
> Le mot de passe du dépôt Restic est indispensable pour accéder aux sauvegardes.
> Si vous le perdez, les données du dépôt deviendront irrécupérables.

4) On peut maintenant initialiser le dépôt, en chargeant d’abord notre script :
```powershell
. $HOME\.restic\restic_env.ps1
restic init
```


## Première sauvegarde {#restic-first-backup}

Avant de lancer une première sauvegarde, quelques explications s’imposent sur le 
fonctionnement de Restic.

Contrairement à une simple copie de fichiers, Restic fonctionne par *snapshots*, 
c’est-à-dire qu’il réalise un instantané de vos données à un instant donné. 
Chaque nouvelle sauvegarde ne transfère ensuite que les données nouvelles ou 
modifiées, ce qui permet de réaliser des sauvegardes incrémentales. On bénéficie 
ainsi d’un historique complet des sauvegardes tout en limitant fortement 
l’espace disque utilisé et le volume de données transférées. La première 
sauvegarde est donc généralement beaucoup plus longue que les suivantes, puisque 
l’ensemble des données doit être copié une première fois.

Il faut savoir également que lors d’une sauvegarde d’un dossier personnel comme 
`C:\Users\julie`, `restic` est parfois dans l’impossibilité d’accéder à certains 
fichiers verrouillés (fichiers de cache et bases de données temporaires utilisés 
par des processus en cours). On peut résoudre ce problème à l’aide de l’option 
`--use-fs-snapshot`, qui permet à `restic` d’utiliser le service VSS de Windows 
pour obtenir un instantané cohérent des fichiers à sauvegarder. À noter que 
cette option impose d’utiliser un [shell](/toolbox/glossary#shell) en mode 
administrateur.

En complément, on exclut quelques dossiers à l’aide de l’option `--exclude` :
- Le dossier `$HOME\AppData\Local\Temp`, qui ne contient que des données 
temporaires sans intérêt pour une restauration.
- Le dossier `$HOME\AppData\Local\Microsoft\WindowsApps`, qui contient 
principalement des alias et fichiers spéciaux recréés automatiquement par 
Windows, et que `restic` n’est pas capable de sauvegarder.

L’usage de ces options permet d’obtenir une sauvegarde sans avertissements sur 
le poste de Julie. Il est cependant possible que sur un autre poste, `restic` 
signale des alertes sur d’autres fichiers spécifiques. Si cela se produit, 
rassurez-vous : ces avertissements n’empêchent pas la sauvegarde de se terminer 
correctement. Ils s’avèrent même utiles au début pour repérer les dossiers 
résiduels et peaufiner ses filtres avant d’automatiser la procédure.

> [!tip] Conseil
> Dans le doute, on peut tester la commande `restic` en ajoutant l’option 
`--dry-run`, qui permet de voir la commande en action sans qu’elle ne touche 
vraiment aux fichiers.

Ceci étant dit, pour procéder à une première sauvegarde, on ouvre un terminal 
PowerShell en mode administrateur, on recharge les variables d’environnement et 
on exécute `restic backup` :
```powershell
. $HOME\.restic\restic_env.ps1
restic backup $HOME `
  --use-fs-snapshot `
  --exclude "$HOME\AppData\Local\Temp" `
  --exclude "$HOME\AppData\Local\Microsoft\WindowsApps"
```

Si tout va bien, en fin de procédure la commande devrait afficher un 
compte-rendu se terminant par :
```text
processed 116513 files, 26.537 GiB in 46:18
snapshot c5391e78 saved
```

Si on veut vraiment en avoir le cœur net, on peut utiliser la commande `restic 
snapshots`, qui donne la liste des sauvegardes effectuées :
```text
repository 7246e2ee opened (version 2, compression level auto)
ID        Time                 Host        Tags        Paths            Size
----------------------------------------------------------------------------------
c5391e78  2026-06-04 13:48:44  jlaptop-av              C:\Users\julie   26.537 GiB
----------------------------------------------------------------------------------
1 snapshots
```


## Automatisation des sauvegardes {#restic-backup-automation}

### Création d’un script de sauvegarde {#restic-backup-script}

Pour automatiser une sauvegarde via Restic dans Windows, il nous faut d’abord 
créer un script PowerShell de sauvegarde :
```powershell
cd $HOME\.restic
micro restic_backup.ps1
```

Voilà un modèle minimal de script que vous pouvez reprendre, améliorer et 
peaufiner à l’aide de la [documentation 
Restic](https://restic.readthedocs.io/en/stable/) :
```powershell
# Load environment
. $HOME\.restic\restic_env.ps1

# Backup
restic backup $HOME `
    --use-fs-snapshot `
    --exclude "$HOME\AppData\Local\Temp" `
    --exclude "$HOME\AppData\Local\Microsoft\WindowsApps"

# Apply retention policy
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

Le script commence par charger les variables d’environnement nécessaires pour 
que Restic travaille avec le bon répertoire, avant de procéder à une sauvegarde 
à l’aide de la commande `restic backup`.

Ensuite, il utilise la commande `restic forget` afin de supprimer les anciens 
snapshots du dépôt. L’option `--prune` permet de supprimer aussi les données 
associées, elle est donc nécessaire pour libérer l’espace disque occupé par les 
anciennes sauvegardes.

> [!info]- Comment définir l’ancienneté des données conservées ?
> La commande proposée dans le script conserve une sauvegarde pour chacun des 
derniers 7 jours, une pour chacune des 4 semaines précédant les 7 jours et une 
pour chacun des 6 mois précédant les 4 semaines. Vous pouvez adapter la 
politique de suppression des anciennes sauvegardes à l’aide des règles suivantes 
:
> - `--keep-daily <X>` : Conserve une sauvegarde journalière pendant les 
derniers `X` jours.
> - `--keep-weekly <Y>` : Conserve une sauvegarde hebdomadaire pendant les 
dernières `Y` semaines.
> - `--keep-monthly <Z>` : Conserve une sauvegarde mensuelle pendant les 
derniers `Z` mois.
> - Vous pouvez même ajouter une règle pour des sauvegardes annuelles 
(`--keep-yearly`) ou peaufiner davantage à l’aide des
[nombreuses options disponibles](https://restic.readthedocs.io/en/stable/060_forget.html#removing-snapshots-according-to-a-policy).
> - À noter que ces règles sont cumulatives. Avec la politique proposée, Restic 
conserve les sauvegardes des 7 derniers jours, puis 4 sauvegardes hebdomadaires 
plus anciennes, puis encore 6 sauvegardes mensuelles.


### Création d’un script d’automatisation {#restic-automation-script}

Pour automatiser la sauvegarde Restic sous Windows, nous écrivons un script 
PowerShell chargé d’enregistrer la tâche dans le Planificateur de tâches Windows 
(Task Scheduler) :
```powershell
cd $HOME\.restic
micro install_sched_task.ps1
```

Voilà un exemple de script permettant d’automatiser la tâche :
```powershell
$Script = "$HOME\.restic\restic_backup.ps1"

$Action = New-ScheduledTaskAction `
  -Execute "pwsh.exe" `
  -Argument "-NoProfile -ExecutionPolicy Bypass -File `"$Script`""

$Trigger = New-ScheduledTaskTrigger `
  -Daily `
  -At 21:00

$Principal = New-ScheduledTaskPrincipal `
  -UserId "$env:USERNAME" `
  -LogonType S4U `
  -RunLevel Highest

$Settings = New-ScheduledTaskSettingsSet `
  -AllowStartIfOnBatteries `
  -DontStopIfGoingOnBatteries `
  -StartWhenAvailable `
  -ExecutionTimeLimit (New-TimeSpan -Hours 4)

Register-ScheduledTask `
  -TaskName "ResticBackup" `
  -Action $Action `
  -Trigger $Trigger `
  -Principal $Principal `
  -Settings $Settings `
  -Description "Daily restic backup to NAS"
```

Ce script crée une tâche automatisée qui exécute `restic_backup.ps1` chaque jour 
à 21h00. Quelques remarques sur les choix de ce script :
- Le mode `S4U` permet à la tâche de s’exécuter avec les droits de l’utilisateur 
sans stocker son mot de passe. La tâche continue donc de fonctionner même si 
l’utilisateur modifie son mot de passe.
- La sauvegarde se produit aussi bien lorsque le laptop est alimenté par secteur 
que par batterie (`-AllowStartIfOnBatteries` et `-DontStopIfGoingOnBatteries`). 
Cela peut être modifié si on souhaite économiser au maximum l’utilisation de la 
batterie. En les supprimant, Windows n’exécutera la tâche que lorsque 
l’ordinateur sera alimenté sur secteur.
- Dans tous les cas, l’option `-StartWhenAvailable` garantit que la sauvegarde 
se fasse dès que possible dans le cas où l’état du laptop ne le permettait pas 
auparavant (si, notamment, il était éteint).
- Le processus de sauvegarde ne peut excéder une durée de 4 heures 
(`-ExecutionTimeLimit (New-TimeSpan -Hours 4)`) afin d’éviter d’avoir un 
processus bloqué indéfiniment dans le cas où il y aurait un problème. Cette 
limite est très largement suffisante sur le poste de Julie puisque la première 
sauvegarde (la seule qui copie l’ensemble des données) a duré environ 45m pour 
26Go. Bien sûr, elle peut être adaptée selon les configurations techniques.

### Activation de la tâche planifiée {#enable-scheduled-task}

Pour enregistrer la nouvelle tâche dans le Planificateur Windows à partir du 
script, il suffit d’exécuter ce dernier dans PowerShell (mode administrateur) :
```powershell
cd $HOME\.restic
.\install_sched_task.ps1
```

On peut vérifier que le script est bien enregistré à l’aide de :
```powershell
Get-ScheduledTask -TaskName ResticBackup
```

On peut également tester un lancement immédiat de la tâche :
```powershell
Start-ScheduledTask -TaskName ResticBackup
```

Après quoi, il est utile de vérifier l’état du dernier run de la tâche :
```powershell
Get-ScheduledTaskInfo -TaskName ResticBackup
```

Ce qui retournera un résultat de ce type :
```text
LastRunTime        : 07/06/2026 22:26:49
LastTaskResult     : 0
NextRunTime        : 08/06/2026 21:00:00
NumberOfMissedRuns : 0
TaskName           : ResticBackup
TaskPath           : 
PSComputerName     : 
```

Ici, `LastRunTime` nous permet de confirmer que la tâche vient effectivement 
d’être accomplie et la valeur de `LastTaskResult` (=0) confirme que le processus 
s’est terminé sans erreurs. Enfin, `NextRunTime` montre que la prochaine 
occurrence interviendra à 21h00 le jour suivant, ce qui est cohérent avec la 
politique définie.

À ce point, autant jeter un œil à l’état du dépôt via `restic snapshots` :
```text
repository 7246e2ee opened (version 2, compression level auto)
ID        Time                 Host        Tags        Paths            Size
----------------------------------------------------------------------------------
c5391e78  2026-06-04 13:48:44  jlaptop-av              C:\Users\julie   26.537 GiB
7784e7dc  2026-06-07 22:26:49  jlaptop-av              C:\Users\julie   26.698 GiB
----------------------------------------------------------------------------------
2 snapshots
```

BOOM !

Tout va bien, voilà un nouveau snapshot qui vient s’ajouter au précédent. En 
comparant les valeurs de la colonne `Size`, on constate que le second snapshot 
contient environ 160 Mo de données supplémentaires par rapport au premier. Cela 
correspond aux fichiers ajoutés ou modifiés entre les deux sauvegardes.

> [!info]- Pour les curieux qui veulent vérifier que les données sont bien là !
> La présence d’un snapshot est déjà un excellent indicateur, mais il est 
possible d’aller plus loin avec quelques commandes simples permettant d’afficher 
le contenu des sauvegardes :
>
> - `restic ls c5391e78` : Pour parcourir le contenu du snapshot `c5391e78`.
> - `restic ls latest` : Pour parcourir le contenu du dernier snapshot.
> - `restic find "exemple.txt" --snapshot latest` : Pour vérifier la présence ou 
chercher le chemin complet du fichier `exemple.txt`.
> - `restic find "\C\Users\julie\Documents" --snapshot latest` : Pour vérifier 
le contenu du dossier personnel `Documents`.
>
> On peut bien sûr aussi immédiatement tester une restauration totale des 
données vers un dossier de test de la manière suivante :
> ```powershell
> cd $HOME
> mkdir restore-test
> restic restore latest --target restore-test
> ```
> 
> Plus simplement, on peut aussi juste tenter de restaurer un dossier ou un 
fichier en ajoutant l’option `--include` :
> ```powershell
> restic restore latest `
>   --target restore-test `
>   --include "/C/Users/julie/Documents/blabla/blibli.PDF"
> ```
> Notez que cette opération restaure dans `$HOME\restore-test\` l’arborescence 
complète du fichier (`C:\Users\julie\Documents\blabla\blibli.PDF`), tout en 
préservant les attributs des fichiers et dossiers. Il se trouve que certains de 
ces dossiers ont l’attribut `readonly` (c’est le cas de `Users\` et 
`Documents\`). Pour effacer tout le dossier de test sans difficultés on peut 
utiliser :
> ```powershell
> Remove-Item $HOME\restore-test\ -Recurse -Force
> ```


## Rclone et Google Drive {#rclone-gdrive-backup}

Il est maintenant temps de mettre en place une sauvegarde complémentaire vers le 
compte Google Drive de Julie via Rclone, ce qui permettra de mieux sécuriser ses 
données.

### Installation et configuration de Rclone (#rclone-installation-windows)

Pour installer Rclone, on ouvre PowerShell en mode administrateur et on passe 
par `winget` :
```powershell
winget install Rclone.Rclone --scope machine
```

On vérifie ensuite que l’exécutable est bien installé et accessible :
```powershell
rclone version
```

<!-- TODO
- faire le ménage dans le PATH (mode GUI)
- rédaction partie rclone+gdrive (le reste)
- supprimer repo GD+software Perfect Backup
- revoir structuration titres autour de ssh+nas / rclone+gdrive
- mise en place procédure surveillance (restic check ?)+option mail -->


### Initialisation du dépot et première sauvegarde

### Automatisation de la sauvegarde

## Surveillance
