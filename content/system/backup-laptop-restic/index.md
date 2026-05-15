+++
title = "Comment j’automatise la sauvegarde de l’ordinateur portable de Madame"
#title = "How to backup your wife’s Windows powered laptop with restic"
date = "2026-05-10T18:54:45+02:00"
draft = true
tags = ["restic", "backup", "windows"]
categories = ["system", "network"]
+++

# Mon article

Ma femme utilise un ordinateur portable sous Windows. Pour elle, l’utilisation
de cet outil doit se limiter au strict minimum (elle préfère, de très loin, son
smartphone). Autant dire que mes encouragements à mettre en place une procédure
de sauvegarde automatique pour éviter de perdre toutes ses données n’ont jamais
donné lieu qu’à des… "Mouais, je verrai ça un jour…" À savoir que pendant
plusieurs années, la seule sauvegarde qu’elle trainait était une vieille clé
USB, solution incertaine au possible.

Désespéré, j’ai fini par entreprendre cette tâche par moi-même, en passant par
des solutions diverses qui ont suivi ma propre courbe d’apprentissage dans le
domaine. D’abord, sauvegarde occasionnelle sur un disque dur externe. Ensuite,
j’ai eu l’idée de tenter de gérer la situation via la solution Onedrive de
Microsoft. On ne m’y reprendra plus. Finalement, je suis passé à un logiciel
(Perfect Backup) permettant des sauvegardes incrémentales et automatiques. Mais
un bug sur une sauvegarde m’en a découragé.

J’ai alors décidé d’aller vers des solutions plus opensource-style avec l’idée
d’avoir quelque chose de solide et (surtout) automatisable pour pouvoir (enfin)
cesser d’y penser. Je vous présente ici les solutions que j’ai construites.

I) Stratégie de backup

Tout d’abord, on va rappeler une règle basique de la sauvegarde personnelle.
Idéalement, un système de backup sérieux doit suivre la règle des 3-2-1...

## Introduction

Bla bla

```bash
#############
# Functions #
#############

# start app function
start_app()
{
    # keep trace of windows list before launching a new app
    list_before=$(wmctrl -l)
    # populate an array containing all ids
    declare -a win_ids
    while read i; do
        win_ids+=($(echo $i | awk '{print $1}'))
    done < <(echo "$list_before")

    # launch app
    if [ "$2" = "null" ]; then $1 2>&1 1>/dev/null &
    else $1 $2 2>&1 1>/dev/null & fi

    # extract app_id from wmctrl
    local app_id
    while [ -z $app_id ]; do
        while read i; do
            if [ $(echo $i | awk '{print $1}') != ">" ]; then continue; fi
            if [ $1 = "xfce4-terminal" ] && \
                [ -z "$(echo $i | grep "Terminal")" ]; then continue
            if [ $(echo $i | awk '{print $1}') != ">" ]; then continue; fieeeeee
[...]
```

## Installation

Bla bla

### SSH

### Restic

## Automatisation

## Conclusion
