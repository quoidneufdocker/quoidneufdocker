---
title: Docker buil 101
date: 2021-11-09T12:12:00+02:00
tags:
  - Build
---

Comme beaucoup d'entre vous, j'ai pesté devant un `docker build` qui télécharge pour la 101ème fois les mêmes paquets `apt install` .

Plutôt que de continuer à subir, je vous propose une série d'articles de mieux comprendre `docker build` et d'explorer quelques pistes pour le rendre plus efficace.

Pour ce premier article, nous allons commencer par les bases.

## back to basics

Un image Docker est une succession de "couches" (_layers_) chacune représentant des modifications apportées au système de fichier. Chaque ligne du `Dockerfile` indique une directive qui va construire une nouvelle couche, qui va venir s'empiler sur la précédente.

Un postulat de Docker, c'est qu'une commande passée sur un état bien défini du système de fichier en cours de construction donnera toujours le même résultat. C'est ce qui lui permet de ne pas rejouer l'intégralité des commandes à chaque `docker build`, mais juste ce qui a été modifié.

Voilà; maintenant, tout ça ne vient pas sans quelques impacts...

## apt-quoi?

Les premières commandes d'un `Dockerfile` sont généralement du type: 

```
FROM ubuntu
RUN apt update && apt install machin bidule chose
```

Il est rarissime de tomber sur un `Dockerfile` dans lequel les dépendances système incluent une version exacte du paquet à installer. C'est bien et pas bien à la fois

1. Si on reconstruit l'image depuis un environnemnt tout neuf, on aura les dépendances système à jour sans devoir courrir après les derniers numéros de version et toucher au Dockerfile.
2. Si on a déjà cette commande en cache, on restera sur des dépendances devenues obsolètes

Il va donc faloir trouver un compromis ... je suggère de faire un `docker build --no-cache` de temps en temps, genre chaque Lundi pendant la pause déjeuner, et je vous laisse le soin de construire un joli workflow de mise à jour sur cette base :P

A titre personnel, je trouverai plus "propre" de mettre des versions aux dépendances système, et d'avoir un outil de type _vulnerability scanner_ qui m'informe qu'un paquet plus récent existe, ou plutôt qui me fait une Pull-Request en mode dependabot pour rafraîchir mon Dockerfile... bref

## t'es là où t'es pas là?

Rien à voir avec [les Tuches](https://www.youtube.com/watch?v=4LyVjNwLabk). Du fait de l'empilement des couches, une _suppression_ n'en est pas réellement une. Un fichier supprimé est bien absent du système de fichier final, mais il existe dans les couches intermédiaires et alourdit l'image.

Résultat, vous verrez un peu partout ce genre de chose:

```
RUN apt-get update \
  && apt-get install -y --no-install-recommends machin 
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*
```

Ce mix de commande installe un paquet système, après avoir mis à jour les métadonnées, puis fait le ménage des divers caches pour ne pas embarquer de fichiers inutiles dans la couche en cours de construction. C'est bien optimisé tout ça, mais bye bye la lisibilibté!

La [syntaxe proposée](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/syntax.md#here-documents) (à titre experimental) par BuildKit sous forme de "[here-document](https://fr.wikipedia.org/wiki/Here_document)" nous économise les `\` et autres `&&` mais bon, le problème reste assez présent:

```
# syntax = docker/dockerfile:1.3-labs
...
RUN <<eof
    apt-get update
    apt-get install -y --no-install-recommends machin 
    apt-get clean 
    rm -rf /var/lib/apt/lists/*
eof    
```

## cache invalidation

L'invalidation des caches est l'un des problèmes les plus complexe de notre industrie, avec le nommage des variables.

Dans le cas d'un `Dockerfile`, `docker build` doit savoir quel layer reconstruire et lequel peut être utilisé tel quel depuis le cache. Là où ça devient compliqué c'est quand il doit ajouter du contentu externe à l'image.

Historiquement nous avions la directive `ADD`. Bon, oubliez là tout de suite. Elle permet de télécharger un truc pour mettre dans l'image, mais du coup, comment savoir s'il faut re-télécharger? Faudrait il respected les headers HTTP "Last-Modified"? Amusez vous à lire dans les issues Docker les kilomètre de débats sur ce sujet.

La commande `COPY` permet d'inclure un fichier local dans l'image. Ici c'est plus simple, si le fichier est le même, le résultat sera (devrait être?) le même, et le cache n'est pas invalidé. S'il est modifié, plus rien n'est sur, le cache est invalide et toutes les commandes à suivre doivent être rejouées.

Et c'est pour ça qu'on évite ce genre de choses: 
```
FROM maven
COPY . /work
RUN mvn package
```

En faisant ça, le moindre changement d'un fichier dans le projet va relancer un build maven complet, avec les 200Mo de téléchargements que cela implique.
On vous recommandra donc des choses nettement plus élaborées, comme:

```
COPY pom.xml /work
RUN mvn dependency:go-offline
COPY . /work
RUN mvn package
```

Cette astuce permet de lier l'étape de téléchargement des dépendances au seul fichier de description du projet (`pom.xml`, `package.json`, `Gemfile`, etc). Tant que celui-ci n'est pas modifié, seul le second `COPY` va invalider le cache. 

Je crois sincèrement que c'est ce qui a propulsé la commande Maven `dependency:go-offline` bien au delà de son cas d'usage initial qui était "je vais bosser dans le TGV" à une époque ou le Wifi à bord tennait de la science fiction.

## to be continued... 

On va en rester là pour ce premier article, il reste beaucoup à dire, mais on va prendre le temps de s'attaquer a chaque sujet séparément.
A+