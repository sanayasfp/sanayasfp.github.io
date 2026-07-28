---
layout: project
title: "Stack : Le gestionnaire d'environnement de dev natif et sans conteneur"
date: 2026-07-28 12:00:00 +0000
lang: fr
tags: [Rust, Tooling, System Design, Backend, CLI]
type: project
github_repo: "https://github.com/sanayasfp/stack"
---

Quand on maintient une application legacy tout en construisant sa remplaçante moderne, l'environnement de développement local devient vite un champ de bataille. Au travail, nous avions une ancienne application tournant sur une vieille version de PHP, Vue 2 (nécessitant une vieille version de Node), et Python 3.7. En parallèle, nous démarrions la nouvelle architecture avec PHP 8.4, Vue 3, et Python 3.12. 

Personnellement, je gérais ça avec une combinaison sur mesure de Laravel Herd pour PHP, `uv` pour Python, et `nvm` pour Node. Ça marchait pour moi parce que j'avais pris le temps de tout configurer. Mais pour nos développeurs juniors et nos nouveaux stagiaires, leur demander de jongler avec ces outils juste pour faire tourner les deux projets en même temps était un cauchemar absolu. Ils passaient leur temps à se battre avec des conflits de chemins et des variables d'environnement au lieu d'écrire du code. C'est pour ça que j'ai créé **Stack** : un gestionnaire d'environnement multi-projets natif, sans conteneur, qui rend le changement de contexte aussi simple que de taper `stack up`.

Le code est ici : [github.com/sanayasfp/stack](https://github.com/sanayasfp/stack).

![Capture d'écran du gestionnaire d'environnement Stack](/assets/stack-screenshot.png)

## La vraie contrainte : l'intégration et la charge cognitive

Avant même de toucher à l'architecture, la vraie contrainte sur un projet comme ça, c'est l'expérience développeur. Les stagiaires ne devraient pas avoir à apprendre le fonctionnement des réseaux bridge de Docker ni à configurer cinq gestionnaires de versions différents pour corriger un bug sur l'API legacy. L'objectif était simple : ils doivent pouvoir cloner le dépôt, lancer une seule commande, et obtenir un domaine local fonctionnel routé vers les bons processus, que le projet utilise PHP 5.6 ou 8.4.

## Pourquoi ne pas juste utiliser Docker ?

Les conteneurs résolvent le problème "ça marche sur ma machine" en embarquant toute une couche OS par projet. Mais cela vient avec le poids de la virtualisation et des démons inactifs qui consomment de la RAM. 

Stack résout le même problème différemment. Il épingle les versions exactes par projet via les gestionnaires de version que vous installeriez de toute façon (en déléguant à des outils comme `vfox` et `uv`), partage un binaire unique téléchargé pour chaque projet nécessitant cette version, et fait tourner le tout sous forme de simples processus enfants. Pas d'hyperviseur, pas de Docker Desktop qui mange 4 Go de RAM pour une API toute simple. Chaque projet qui épingle le même service et la même version partage une instance unique en cours d'exécution, isolée par schéma.

## Sous le capot : la simplicité avant la magie

Un fichier `stack.toml` dans votre projet définit exactement ce dont il a besoin :

```toml
[project]
name = "acme-api"

[language]
php = "8.4.0"

[service.mysql]
version = "8.0.35"

[run]
command = "php -S 127.0.0.1:{port} -t public"
```

Quand vous tapez `stack up`, l'orchestrateur lit le manifeste, configure le domaine local (par ex. `acme-api.localhost`), démarre MySQL s'il ne tourne pas déjà, lance le serveur de dev PHP, et fait le proxy inverse du trafic. Tout est natif, et incroyablement rapide. Il n'essaie pas de réinventer la gestion des paquets ; il délègue le gros du travail de téléchargement des langages à `vfox` et `uv`, et le routage à Caddy.

## Où en est le projet actuellement

Je tiens à être précis là-dessus : j'ai commencé à coder Stack en Rust la semaine dernière seulement, et j'ai poussé la première version il y a deux jours. J'ai complètement désinstallé Herd, Laragon et mes précédents scripts maison. J'utilise désormais exclusivement Stack pour mon travail quotidien — ce qui me permet de prouver que ça marche, de trouver des bugs dans les cas limites, et d'ajouter de nouvelles fonctionnalités pour l'améliorer. 

Il supporte actuellement Windows (PowerShell & cmd), ce qui était le besoin immédiat, avec macOS et Linux prévus sur la feuille de route. Ce n'est plus juste une expérience de week-end ; c'est en train de devenir l'outil fondamental qui garde notre équipe saine d'esprit côté dev local.

Vous pouvez récupérer l'installateur sur le dépôt ou lire la documentation officielle :
- Dépôt : [github.com/sanayasfp/stack](https://github.com/sanayasfp/stack)
- Docs : [sanayavo.com/stack/](https://sanayavo.com/stack/)
