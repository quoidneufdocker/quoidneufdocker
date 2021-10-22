---
title: Docker build telemetry
date: 2021-10-22T12:12:00+02:00
tags:
  - Build
---


> Garde toujours tes amis près de toi et tes enemis, encore plus près.

-- Al Pacino 

Comme beaucoup d'entre vous, j'ai pesté devant un `docker build` qui télécharge pour la 101ème fois les mêmes dépendances lors d'un `apt install` ou d'un `npm install` après avoir ajouté une ligne dans mon `package.json`.
