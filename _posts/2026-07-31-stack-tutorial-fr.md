---
layout: post
title: "Le Guide Ultime pour un Environnement de Dev Fluide"
date: 2026-07-31 09:00:00 +0000
lang: fr
tags: [Rust, Outils, PHP, System Design, Backend, CLI]
excerpt: "Fatigué que Docker fasse fondre votre ordinateur ? J'ai passé une semaine à perfectionner Stack pour que vous n'ayez pas à le faire. Voici le guide ultime, étape par étape, pour un environnement PHP + MySQL sans friction."
---

J'ai [écrit au sujet de Stack]({% link _posts/2026-07-28-stack-dev-env-en.md %}) il y a quelques jours — pour vous donner la version courte de pourquoi je l'ai créé et de toute la philosophie "sans Docker". Mais la théorie a ses limites. Si vous cherchez le guide ultime pour obtenir un environnement de développement parfaitement fluide, vous êtes au bon endroit.

Pas de pitch commercial. Pas de blabla marketing. Juste la réalité brute et pratique de la mise en route d'un projet sans effort, avec chaque invite de commande et chaque sortie console un peu bizarre que vous rencontrerez en cours de route.

Si vous voulez juste la documentation de référence brute, vous la trouverez sur [sanayavo.com/stack](https://sanayavo.com/stack/). Sinon, bienvenue dans la visite guidée d'un meilleur flux de travail (workflow).

## Ce que vous obtenez vraiment

Un environnement fluide signifie zéro superflu. Stack n'est qu'un seul binaire Rust. Il n'y a pas de démon qui monopolise la RAM dans votre barre des tâches, pas de Docker Desktop qui fait tourner les ventilateurs de votre ordinateur portable à fond, et absolument rien qui ne touche à votre installation PHP globale du système.

Au lieu de cela, il agit comme du ruban adhésif (duct tape) hautement intelligent. Il s'appuie sur [vfox](https://github.com/version-fox/vfox) pour gérer les versions de PHP/Node, [uv](https://github.com/astral-sh/uv) pour Python, et [Caddy](https://caddyserver.com) pour des domaines locaux instantanés. Stack lit un simple fichier `stack.toml` à la racine de votre projet et connecte automatiquement ces trois outils ensemble.

*Note : Stack est uniquement disponible sur Windows pour le moment (PowerShell ou cmd). Le support de Mac et Linux est prévu sur la feuille de route.*

## L'installation : Votre seul et unique mal de crâne

Un flux de travail quotidien fluide nécessite un tout petit peu de configuration au départ. Exécutez ceci pour télécharger l'installateur :

```powershell
irm https://github.com/sanayasfp/stack/releases/latest/download/stackenv-installer.ps1 | iex
```

Ensuite, exécutez cette commande *une seule fois* par machine :

```powershell
stack setup
```

Cela s'occupe du gros du travail en arrière-plan. Cela ajoute un hook (crochet) à votre profil PowerShell (pour que la bonne version de PHP ou Node s'active par magie lorsque vous faites un `cd` dans un répertoire), installe vfox, uv et Caddy dans des versions largement testées, et exécute `caddy trust`. Cette dernière étape est cruciale pour le HTTPS local — on y revient dans une minute.

```console
$ stack setup
added the stack hook for pwsh
checking vfox/uv/caddy...
  vfox: OK (1.0.11)
  uv: OK (0.11.7)
  caddy: OK (2.11.4)
  caddy: local CA trusted (https://*.localhost works with no browser warning)
```

**Redémarrez votre terminal maintenant.** Si vous sautez cette étape, le hook PowerShell ne sera pas actif et vous vous demanderez pourquoi rien ne fonctionne. Vous n'aurez plus jamais à exécuter `stack setup` à moins d'acheter un nouvel ordinateur.

## Démarrer un projet sans friction

Construisons quelque chose de concret pour vous montrer le fonctionnement : une API PHP légère adossée à MySQL.

### À partir de zéro

```console
$ stack new acme-api
domain: acme-api.localhost
languages (space to toggle, enter to confirm): [x] php
services (space to toggle, enter to confirm): [x] mysql
  php version: 8.3.1
  mysql version: 8.0.35
created acme-api\stack.toml
next: cd into it, add a [run] command when you know it, then `stack up`
```

C'est une liste de contrôle interactive dans le terminal. Espace pour basculer (cocher/décocher), Entrée pour confirmer. Ça fait un peu bizarre la première fois, mais rien n'est validé tant que vous n'appuyez pas sur Entrée. Si vous sélectionnez accidentellement Node au lieu de PHP, appuyez à nouveau sur espace. Zéro stress, pas besoin de tout recommencer.

### Si vous avez déjà un projet

Si votre dossier `acme-api` possède déjà un fichier `composer.json` spécifiant `"php": "^8.3"`, Stack est assez intelligent pour vous éviter de taper manuellement :

```console
$ stack init
detected from existing project files:
  php 8.3 (from composer.json)
domain: acme-api.localhost
```

`stack init` analyse automatiquement `composer.json`, `package.json` ou `pyproject.toml` et pré-remplit l'assistant pour vous.

### L'artefact magique

Stack génère un manifeste très lisible :

```toml
[project]
name = "acme-api"
domain = "acme-api.localhost"

[language]
php = "8.3.1"

[service.mysql]
version = "8.0.35"
```

Commitez ce fichier `stack.toml` dans votre dépôt (repo). C'est votre artefact ultime du type "ça marche sur ma machine" — l'équivalent léger et sans conteneur d'un Dockerfile. Lorsqu'un collègue clone le dépôt et tape `stack up`, il obtient instantanément les mêmes versions exactes de PHP et MySQL, sans avoir à installer quoi que ce soit à la main.

## Lui apprendre comment s'exécuter

Votre `stack.toml` doit savoir comment démarrer votre serveur de développement. Ouvrez le fichier et ajoutez un bloc `[run]` :

```toml
[run]
command = "php -S 127.0.0.1:{port} -t public"
```

L'espace réservé `{port}` est dynamiquement remplacé par un port disponible au moment de l'exécution. (Note : Si vous déclarez `[language.php]` mais ignorez la section `[run]`, Stack se rabat sur sa propre configuration FastCGI intégrée utilisant `php-cgi.exe` et Caddy, ce qui est incroyablement solide. Mais pour ce guide, nous allons rester explicites.)

## `stack up` et regardez la magie opérer

```console
$ cd acme-api
$ stack up
Loaded C:\Users\you\acme-api\stack.toml
  project: acme-api
  domain: acme-api.localhost
  languages: php
  services: mysql

first run for this project — stack.toml will execute:
  [run] php -S 127.0.0.1:{port} -t public
Trust and run these commands? [y/N] y

  php: C:\Users\you\.vfox\cache\php\v-8.3.1\...\php.exe -> PHP 8.3.1 (cli)

  service.mysql: started (pid 41232, port 3306)
    schema 'acme_api' — automatic creation not yet implemented; create it manually if needed

  run: php -S 127.0.0.1:52140 -t public  (pid 41244, port 52140)
  log: C:\Users\you\.stack\logs\acme-api.log
  routed: http://acme-api.localhost -> 127.0.0.1:52140
```

Il y a deux choses majeures qui se passent ici et qui contribuent à une expérience sans accroc :

### 1. L'invite de confiance de sécurité

Parce que `[run].command` est une invocation littérale du shell, Stack l'exécute exactement tel qu'il est écrit. La première fois que Stack voit une commande nouvelle ou modifiée (comme après un `git pull`), il vous demande de la confirmer. Dites *oui*, et il mémorise votre choix dans `~/.stack/trust.json`. Chaque `stack up` suivant sera heureusement silencieux. Besoin de contourner cela pour la CI ? Passez simplement le flag `--yes`.

### 2. Le HTTPS fonctionne immédiatement

La console affiche `http://`, mais `https://acme-api.localhost` fonctionne instantanément. Aucun avertissement rouge du navigateur. Vous vous souvenez de cette commande `caddy trust` lors de la phase de configuration ? Elle installe une autorité de certification locale (similaire à `mkcert`), ce qui signifie que votre environnement local reflète fidèlement le SSL de production dès le premier jour.

## Ce que "sans conteneurs" signifie vraiment en pratique

Ouvrez un deuxième terminal, faites un `cd` dans un projet totalement *différent* qui nécessite également MySQL 8.0.35, et exécutez `stack up` :

```console
  service.mysql: already running, shared with other projects (pid 41232, port 3306)
```

C'est toute la beauté de la chose. Un seul processus MySQL sert deux projets, isolés par schéma plutôt que de brûler votre processeur et votre RAM avec des conteneurs redondants.

Mais c'est ici que le facteur "fluide" brille vraiment. Ouvrez un troisième terminal, faites un `cd` dans `acme-api`, et exécutez simplement `php -v`. Pas de `stack up`, pas de commandes supplémentaires :

```console
$ cd acme-api
$ php -v
PHP 8.3.1 (cli) (built: ...)
$ cd ..
$ php -v
'php' is not recognized as an internal or external command
```

Le changement de version se fait de manière *ambiante* à chaque invite de commande grâce à ce hook PowerShell. Votre IDE, `composer install` et vos exécuteurs de tests obtiennent tous automatiquement la bonne version épinglée à la seconde où ils entrent dans le dossier. Quittez le répertoire, et elle disparaît. Pas de `source venv/bin/activate`, pas de `nvm use`, et pas de pollution de votre système global.

## Des installations PHP toutes neuves qui ne craignent pas

Habituellement, une nouvelle installation PHP signifie passer une après-midi à traquer les erreurs "PDO driver not found" ou "timezone not set". Stack élimine complètement cela.

La première fois qu'il télécharge une version de PHP via vfox, il patche automatiquement `php.ini`. Il active OPcache, augmente les limites de mémoire et de téléchargement (upload), définit un fuseau horaire, et active les extensions dont vous avez réellement besoin par défaut (`pdo_mysql`, `pdo_pgsql`, `pdo_sqlite`, `sockets`, `sodium`, etc.). Cela se fait une fois, automatiquement, et vous n'avez plus jamais à y penser.

## `stack doctor` — L'ultime vérification avant le vol

Avant de transmettre un projet à un collègue ou de commencer à déboguer un problème étrange, exécutez ceci :

```console
$ stack doctor --project
checking C:\Users\you\acme-api\stack.toml...
  language.php: OK (C:\Users\you\.vfox\cache\php\v-8.3.1\...\php.exe)
  service.mysql: OK (managed)
```

Cette commande valide vos ports, chemins et valeurs `{PLACEHOLDER}` par rapport à votre environnement (en chargeant d'abord `.env`) sans démarrer un seul service. Si quelque chose est cassé, vous obtenez une liste claire dès le départ au lieu de découvrir des erreurs cryptiques au beau milieu du démarrage.

## Pilotez votre configuration de n'importe où

Une fois que Stack connaît un projet, vous n'avez même pas besoin d'être dans son dossier pour le gérer :

```console
stack describe acme-api
stack restart acme-api
stack down acme-api
```

`stack describe` affiche tout — les chemins binaires résolus, l'emplacement exact de votre `php.ini` haché, les logs, et les domaines routés. Stack conserve un registre local de vos projets, ce qui rend la gestion globale sans effort.

## Tout fermer proprement

Quand la journée est terminée :

```console
stack down --all
```

Tout s'arrête. Chaque projet, chaque service partagé, et Caddy. Vous obtenez une utilisation CPU vraiment à zéro, et non un conteneur fantôme persistant que vous avez oublié de tuer.

## Une petite note sur les domaines personnalisés

L'utilisation de `.localhost` garantit une expérience fluide car les navigateurs le résolvent automatiquement vers l'adresse de bouclage (RFC 6761) sans que vous ayez besoin de bidouiller votre fichier hosts. Cependant, si vous migrez depuis Laragon ou Herd et que vous avez absolument besoin de domaines `.test`, il y a une configuration unique à faire en utilisant [Acrylic DNS Proxy](https://sanayavo.com/stack/custom-domains.html).

## Votre Nouveau Flux de Travail

Générez (Scaffold), exécutez, faites confiance une fois, profitez du HTTPS automatique, commitez votre manifeste, et détruisez tout proprement quand vous avez terminé.

J'utilise ce flux de travail exact pour du vrai travail client chaque jour. Ce n'est pas un scénario idéal théorique ; c'est un plan d'action éprouvé pour un environnement de développement ultime et fluide.

Pour la référence complète de `stack.toml`, les options de ligne de commande (CLI), et les fonctionnalités avancées (comme l'utilisation de `[[clone]]` pour démarrer à partir d'un simple manifeste), consultez la documentation officielle :

- [Pourquoi j'ai créé cela](https://sanayavo.com/stack/why.html) — la longue diatribe contre Docker/XAMPP pour le dev local
- [Pour commencer](https://sanayavo.com/stack/getting-started.html)
- [Référence du Manifeste](https://sanayavo.com/stack/manifest.html)
- [Référence CLI](https://sanayavo.com/stack/cli.html)
- Code : [github.com/sanayasfp/stack](https://github.com/sanayasfp/stack)
