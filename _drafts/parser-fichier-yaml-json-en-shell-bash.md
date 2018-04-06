---
layout: post
title:  "Comment parser des fichiers yaml/json en shell grace à Niet"
description: ""
category: [ shell, yaml, parsing, bash, GNU/linux ]
date:   2018-4-6 00:00:00 +0100
image: port-knocking.jpg
---
## Introduction
Dans cet article je vais tenter d'expliquer comment faire pour parser des fichiers
YAML/JSON directement dans votre shell (bash, sh, etc...) ou dans vos scripts shell.

## Prérequis
- Connaitre les bases du shell et comment jouer des commandes
- Connaitre les bases de la syntaxe JSON et YAML

## Problème
Il peut arriver qu'un jour vous ayez besoin de récupérer des valeurs contenues 
dans un fichier YAML ou JSON depuis votre shell, hors le shell GNU/Linux ne permet
se genre de manipulations nativement, il vous faudra passer par un outil tiers
pour arriver à vos fins.

Dans cette article nous allons présenter les différentes solutions qui s'offrent à vous
pour réaliser des extractions de données depuis vos fichiers YAML afin de les
utiliser directement dans vos commandes shell ou vos scripts shell.

## Solution
Il existe plusieurs solutions et plusieurs outils pour extraire vos données depuis
vos fichiers YAML/JSON, dans cet article je vais en présenter un en particulier, ce
dernier se nomme [*niet*](https://github.com/gr0und-s3ct0r/niet/).

Niet est un petit utilitaire utilisable en ligne de commande. Il écrit en Python et il est disponible via [PyPi](https://pypi.org/project/niet/).

J'ai choisi de vous présenter niet pour plusieurs raisons :
- sa simplicité d'utilisation qui suffira largement dans la majorité des cas;
- j'en suis l'auteur et j'aime partager mon travail et mes connaissances.

## Installer ou mettre à jour niet
Niet étant disponible sur PyPi (Python Packages Index), ce dernier peut être
installé par n'importe quel package manager de l'écosystème Python comme `pip` notamment.

```shell
# l'option -U permet de mettre à jour niet si ce dernier est déjà installé
$ pip install -U niet
```

## Utiliser niet
Niet peut être utilisé aussi bien en ligne de commande qu'au sein d'un script shell.

Vous pouvez aisement interfacer votre code shell avec les retours de ce dernier
afin de procéder à des traitements et des actions spécifique sur vos données.

Imaginons le fichier YAML suivant (`sample.yaml`) :
```shell
$ cat sample.yaml
# sample.yaml
project:
    meta:
        name: project-sample
        tags:
          - example
          - sample
          - for
          - testing
          - purpose
```

Vous pouvez récupérer les valeurs de ce dernier en l'utilisant de la manière suivante :
```shell
$ niet sample.yaml project.meta.name
project-sample
```

### Travailler avec des listes de valeurs
Un usage intéressant serait de vouloir exécuter des actions répetées
depuis votre ligne de commande.

Par exemple imaginons que vous souhaitiez installer une liste de RPM définis
dans un fichier de configuration au format YAML, considérons donc le fichier
YAML suivant :

```shell
$ cat config.yaml
# sample.yaml
packages:
    - docker
    - ansible
    - tmux
```

Grâce à niet vous pourrez installer vos paquets de la manière suivante :

```shell
$ for package in $(niet config.yaml packages); do yum install -y  ${package}; done
...
$ # les paquets docker, ansible, et tmux viennent d'être installés
```

### Utiliser niet dans un script shell
Bien évidemment vous pouvez utiliser `niet` au sein de vos scripts shell.

Il s'utilise de la même manière qu'en ligne de commande (CLI).

Voici le même exemple que celui-ci dessus mais avec une syntaxe plus orienté scripting :

```shell
PACKAGES=$(niet config.yaml packages)
if [ "$?" = "1" ]; then
    # Affiche le message d'erreur suivant
    # Error occur! Element not found: packages
    echo "Error occur: ${PACKAGES}"
    exit 1
else
    for package in ${PACKAGES};
    do 
        yum install -y ${package}
    done
    exit 0
fi
```

### Gérer les erreurs
Si le fichier n'existe pas ou n'est pas trouvé par défaut `niet` vous retournera
le code d'erreur `1` et vous affichera le message d'erreur suivant `Yaml file not found! Abort!`

Si la clé n'existe pas ou n'est pas trouvé `niet` vous retournera également un
code de retour égale à `1` et le message suivant s'affichera `Element not found: votre.cle`

## Alternatives à niet
Il existe d'autres utilitaires en ligne de commande comme l'excellent 
[jq](https://stedolan.github.io/jq/) notamment.

### JQ
Ce dernier est très complet en terme de fonctionnalités par contre.
Jq vous offre la possibilité de lire vos données via une syntaxe de type [xpath](https://fr.wikipedia.org/wiki/XPath)
par contre il vous autorise uniquement la manipulation de fichiers au format JSON.

### JyParser
[Jyparser](https://github.com/jlordiales/jyparser) est capable de lire du YAML et du JSON
par contre son utilisation se fait au travers d'un [container docker](https://hub.docker.com/r/jlordiales/jyparser/builds/)
son utilisation nécessite donc la mise en place d'un environnnement docker, ce qui
je trouve est relativement lourd comme solution pour un usage aussi basic (c'est mon avis personnel).

## Conclusion
Niet couvrira une bonne partie de vos besoins quotidien en terme de lecture de fichiers
YAML/JSON en shell. Il présente l'avantage d'être léger et suffisament puissant
pour couvrir des usages relativement complexes. 

Le fait que Python soit installé sur l'ensemble des distributions GNU/Linux vous permet
de l'installer facilement et de manière quasi transparente.

En espérant vous avoir convaincu!

