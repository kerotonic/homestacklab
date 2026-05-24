+++
title = "Glossaire"
date = "2026-05-23"
lastmod = "2026-05-24"
draft = false
tags = ["toolbox"]
+++

#### Shell
Le shell est l’interface textuelle qui permet de communiquer avec le système 
d’exploitation d’une machine. Contrairement à une interface graphique (où l’on 
clique sur des icônes), le shell attend que l’utilisateur saisisse des commandes 
au clavier pour les exécuter.

On utilise généralement le shell à travers un terminal : une fenêtre permettant 
d’afficher le shell et d’interagir avec lui, comme l’illustre l’exemple 
ci-dessous :

{{< figure
  src="images/shell.png"
  alt="Shell example"
>}}

#### SSH (clé)
Une clé SSH est un mécanisme d’authentification permettant de se connecter à un 
[serveur SSH](#ssh-serveur) sans que celui-ci ne demande de mot de passe. Elle fonctionne 
avec une paire de clés :
- une clé privée, conservée secrètement sur votre machine
- une clé publique, copiée sur le serveur distant

Lors de la connexion, le serveur vérifie que vous possédez bien la clé privée 
correspondant à la clé publique enregistrée.

#### SSH (serveur)
Un serveur SSH est un programme qui tourne en arrière-plan sur une machine 
distante (serveur, Raspberry Pi, NAS…) et qui permet à un utilisateur de s’y 
connecter afin d’exécuter des commandes sur cette machine. Typiquement, une 
connexion SSH ouvre un [shell](#shell) distant permettant de piloter le système en ligne 
de commande.
