+++
title = "Vim : petit guide de survie"
description = "Le strict minimum pour ne plus paniquer quand vous êtes parachuté dans Vim."
date = "2026-05-24"
lastmod = "2026-05-24"
draft = false
tags = ["toolbox", "vim", "linux"]
+++

&nbsp;

> [!note] Note
> Pour voir immédiatement les principales commandes de survie, c’est [par 
là](#main-commands).

Vim est un éditeur en mode texte extrêmement populaire dans le monde du libre en 
raison de sa puissance et de sa polyvalence. Mais son utilisation peut dérouter 
le néophyte puisqu’elle est contre-intuitive par rapport aux autres éditeurs de 
texte. C’est ce qui explique que, si certains ne jurent que par lui une fois 
qu’ils ont suivi la courbe d’apprentissage permettant d’en maîtriser la 
puissance, d’autres conservent un rapport ambigu avec « le machin ».

Il demeure que connaître quelques (petites) notions sur Vim est, tôt ou tard, 
indispensable si on fait de la gestion informatique domestique, ne serait-ce que 
parce que nombre de systèmes embarqués ne proposent que Vim comme interface 
d’édition lorsqu'on s'y connecte en SSH pour modifier des fichiers de 
configuration système. C’est le cas de mon NAS Synology ou de mon routeur 
UCG-Ultra.


## Les 5 commandes pour survivre dans un univers inconnu {#main-commands}

Quand vous ouvrez Vim, que vous êtes perdus parce que vous ne savez pas quoi 
faire, les commandes suivantes vous permettent d’être immédiatement 
opérationnel :
- `i` : Pour commencer à écrire ou modifier du texte (passage en mode 
*Insertion*).
- `Échap` ou `ESC` : Pour arrêter le travail d’écriture (retour au mode 
*Commande*).
- `:w` : Pour sauvegarder votre travail.
- `:q` : Pour quitter (si tout est sauvegardé).
- `:q!` : Pour quitter en force sans sauvegarder.

> [!note] Note
> Les commandes qui commencent par "`:`" s'exécutent en pressant `Entrée`.


## Les commandes quand vous vous dites que maintenant, vous n’avez plus peur de rien {#beyond-survival}

Une fois le choc initial passé, manipuler un fichier de configuration sera 
facilité si vous disposez de quelques outils supplémentaires. Voici le package « 
confort » pour ne pas y passer des heures.

> [!note] Note
> Sauf indication contraire, toutes les combinaisons suivantes s’effectuent en 
mode *Commande*.


### Pour vous déplacer {#navigation}

Les flèches directionnelles (`←  ↑  →  ↓`) et les touches `Début`, `Fin`, `Page 
Préc.` et `Page Suiv.` fonctionnent normalement (en mode *Commande* comme en 
mode *Insertion*). En mode *Commande* vous pouvez aussi utiliser les 
combinaisons suivantes qui vous permettent de ne pas éloigner les doigts de la 
zone d’écriture :
- `h` / `j` / `k` / `l` : Gauche / Bas / Haut / Droite.
- `gg` : Aller directement au tout début du fichier.
- `G` : Aller tout à la fin du fichier.


### Pour éditer {#editing}

Pour effacer ou corriger proprement, utilisez les touches suivantes :
- `Retour arrière` (Backspace) : Efface le caractère avant le curseur 
(uniquement en mode *Insertion*).
- `Suppr` (Delete) : Efface le caractère après le curseur (en mode *Commande* et 
*Insertion*).
- `x` : Supprime le caractère sous le curseur (l'équivalent de la touche 
`Suppr`).
- `dd` : Supprime (coupe) la ligne entière où se trouve le curseur.
- `u` : Annule la dernière action.
- `Ctrl + r` : Rétablit l'action annulée.


### Pour rechercher {#search}

Pour rechercher un mot, suivez les étapes suivantes :
1) Tapez `/` puis la chaîne à rechercher (par exemple 
`/PasswordAuthentication`).
2) Tapez `Entrée` pour lancer la recherche. Vim placera le curseur sur la 
première occurrence trouvée.
3) Tapez `n` pour passer à l’occurrence suivante (ou `N` pour revenir à la 
précédente).


### Pour copier-coller {#copy-paste}

Dans Vim, le copier-coller est un concept complexe sur lequel on pourrait écrire 
une thèse de doctorat. Mais cette procédure vous permettra de toujours vous en 
sortir :
1) Placez le curseur au début ou à la fin du texte à copier ou couper et appuyez 
sur `v`.
2) Amenez le curseur à l’extrémité inverse du texte à copier/couper (à la fin ou 
au début, respectivement).
3) Ensuite, si vous voulez :
- copier : appuyez sur `y`
- couper : appuyez sur `d`
4) Amenez le curseur là où vous souhaitez coller et appuyez sur : `p`. Notez que 
le texte sera collé *après* le curseur.


### Pour copier-coller en trichant {#clipboard-paste}

Une situation régulière qui se produira est que vous souhaitez coller dans Vim 
un texte copié en dehors de Vim (typiquement, une ligne de code ou de 
configuration trouvée dans un howto ou un forum, que vous souhaitez coller dans 
une instance de `vim`, elle-même ouverte dans un terminal graphique, par exemple 
xfce4-terminal). Dans ce cas, vous pouvez utiliser la fonction « copier » du 
terminal :
1) En dehors de Vim, copiez le texte qui vous intéresse via la combinaison 
standard `Ctrl + c`.
2) Dans Vim, placez le curseur là où vous souhaitez coller et passez en mode 
*Insertion* (`i`).
3) Appuyez sur `Ctrl + Maj + v` (valable pour xfce4-terminal, Gnome Terminal, 
Konsole… À vérifier pour le vôtre).

> [!tip] Astuce
> Si un texte collé de cette manière se décale étrangement « en escalier », 
tapez `u` (pour annuler), puis la commande `:set paste`. Collez votre texte, 
puis tapez `:set nopaste` une fois terminé.
