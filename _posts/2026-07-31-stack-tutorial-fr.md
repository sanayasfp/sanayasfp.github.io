---
layout: post
title: "Le guide complet de Stack : faire tourner plusieurs projets PHP, Node et Python sans Docker"
date: 2026-07-31 09:00:00 +0000
lang: fr
tags: [Rust, Tooling, PHP, System Design, Backend, CLI]
excerpt: "Une prise en main réelle de Stack : créer un projet PHP+MySQL, comprendre stack.toml, la confirmation à la première exécution, le HTTPS automatique, et les commandes du quotidien, de bout en bout."
---

J'ai écrit un [court article]({% link _posts/2026-07-28-stack-dev-env-fr.md %}) il y a quelques jours pour présenter **Stack** — pourquoi je l'ai créé, et l'idée centrale (des processus natifs plutôt que des conteneurs). Celui-ci est différent : c'est le tutoriel que j'aurais aimé trouver la première fois que j'ai essayé l'outil. Pas de discours, juste le vrai flux de travail, commande par commande, sur un vrai projet PHP + MySQL, avec chaque sortie affichée exactement comme Stack l'imprime aujourd'hui (v0.1.26).

Si vous cherchez juste la référence technique, elle est sur [sanayavo.com/stack](https://sanayavo.com/stack/) — cet article en est la visite guidée.

## Ce que vous installez réellement

Stack est un unique binaire Rust. Il ne fait tourner aucun démon, n'a pas besoin de Docker Desktop, et ne touche pas à votre installation système de PHP/Node/Python. Il délègue la gestion des versions à deux outils que vous installeriez probablement de toute façon — [vfox](https://github.com/version-fox/vfox) pour PHP/Node, [uv](https://github.com/astral-sh/uv) pour Python — et route les domaines locaux via [Caddy](https://caddyserver.com). Le travail de Stack se résume à faire coller ces trois outils autour d'un seul fichier par projet : `stack.toml`.

Windows est la seule plateforme supportée pour l'instant (PowerShell et cmd). macOS et Linux sont prévus, pas encore livrés.

## Installation et configuration initiale

```powershell
irm https://github.com/sanayasfp/stack/releases/latest/download/stackenv-installer.ps1 | iex
```

Puis, une seule fois, sur cette machine :

```powershell
stack setup
```

Cette commande fait deux choses : elle installe un hook dans votre profil PowerShell (pour que la chaîne d'outils épinglée d'un projet s'active automatiquement à chaque `cd` dedans), et elle installe vfox, uv et Caddy aux versions avec lesquelles Stack est testé. Elle lance aussi `caddy trust` — j'y reviens dans un instant.

```console
$ stack setup
added the stack hook for pwsh
checking vfox/uv/caddy...
  vfox: OK (1.0.11)
  uv: OK (0.11.7)
  caddy: OK (2.11.4)
  caddy: local CA trusted (https://*.localhost works with no browser warning)
```

Redémarrez votre terminal une fois pour que le hook prenne effet. Vous n'aurez plus besoin de relancer `stack setup`, sauf en changeant de machine.

## Votre premier projet

Construisons quelque chose de réel : une petite API PHP adossée à MySQL. Deux façons de démarrer, selon que le projet existe déjà ou non.

### Partir de zéro

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

Le choix des langages et services se fait via une liste à cases à cocher — espace pour cocher, entrée pour confirmer. Rien n'est écrit avant votre confirmation : cocher la mauvaise case par erreur se corrige en la décochant, ce n'est pas une raison de tout recommencer.

### Partir d'un projet existant

Si `acme-api` a déjà un `composer.json` contenant `"php": "^8.3"`, inutile de tout retaper à la main :

```console
$ stack init
detected from existing project files:
  php 8.3 (from composer.json)
domain: acme-api.localhost
```

`stack init` lit `composer.json`, `package.json` ou `pyproject.toml` et pré-remplit les contraintes de version qu'il y trouve. Le même assistant s'enchaîne ensuite.

### Lire le manifeste généré

```toml
[project]
name = "acme-api"
domain = "acme-api.localhost"

[language]
php = "8.3.1"

[service.mysql]
version = "8.0.35"
```

Ce fichier est fait pour être commité. C'est tout l'artefact nécessaire pour "partager un projet" — l'équivalent sans conteneur d'un Dockerfile. Quiconque clone le dépôt et lance `stack up` obtient exactement la même version de PHP et exactement la même version de MySQL, sans installer ni l'un ni l'autre à la main.

## Lui dire comment se lancer

`stack.toml` ne connaît pas encore votre commande de serveur de dev. Ouvrez-le et ajoutez une section `[run]` :

```toml
[run]
command = "php -S 127.0.0.1:{port} -t public"
```

`{port}` est remplacé par un vrai port alloué au démarrage. Si vous omettez `[run]` entièrement et que `[language.php]` est déclaré, Stack bascule par défaut sur son propre moteur FastCGI (`php-cgi.exe` avec de vrais processus de travail concurrents, exposé via une route FastCGI de Caddy) plutôt que d'exiger une commande explicite — le même schéma de "valeur par défaut sensée" que les services ont déjà. On y revient plus bas.

## Le démarrer

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

Deux détails méritent qu'on s'y arrête, parce qu'ils ne sautent pas aux yeux sur une simple capture d'écran.

### La demande de confirmation

`[run].command` et chaque `[service.*].command` sont des invocations de programme littérales — Stack les exécute exactement telles quelles, de la même façon que `npm install` exécute ce qu'il trouve dans les scripts de `package.json`. La première fois que les commandes d'un projet sont vues, ou la première fois qu'elles changent (par exemple après un `git pull` qui en apporte une différente), Stack les affiche et demande confirmation avant d'exécuter quoi que ce soit. Refusez, et rien ne démarre. Approuvez une fois, et c'est mémorisé dans `~/.stack/trust.json` — chaque `stack up` suivant sur ce même jeu de commandes est silencieux. Passez `--yes` pour sauter la confirmation en CI ou dans un script.

### C'est déjà en HTTPS

Cette ligne `routed:` n'affiche que `http://`, mais `https://acme-api.localhost` fonctionne déjà aussi, sans aucun avertissement du navigateur. L'étape `caddy trust` de `stack setup` a installé l'autorité de certification locale de Caddy dans le magasin de confiance de votre système — mécaniquement la même chose que fait [mkcert](https://github.com/FiloSottile/mkcert). Rien ne force la redirection ; les deux protocoles sont actifs sur la même route.

## Ce que "sans conteneur" apporte concrètement ici

Ouvrez un second terminal, faites un `cd` vers un *autre* projet qui déclare aussi `mysql = "8.0.35"`, et lancez `stack up` là aussi :

```console
  service.mysql: already running, shared with other projects (pid 41232, port 3306)
```

Un seul processus MySQL, deux projets, isolés par schéma plutôt que par un conteneur séparé chacun. Si vous faites `cd acme-api` dans un troisième terminal, là, maintenant, et que vous lancez juste `php -v` à la main — sans `stack up`, sans commande du manifeste — vous obtenez 8.3.1, parce que l'activation se fait de manière ambiante à chaque invite, pas seulement à l'intérieur de `stack up` :

```console
$ cd acme-api
$ php -v
PHP 8.3.1 (cli) (built: ...)
$ cd ..
$ php -v
'php' is not recognized as an internal or external command
```

C'est exactement le point de Stack : `composer install`, un lanceur de tests, l'interpréteur PHP configuré dans votre IDE — tout ce qui exécute `php` depuis ce dossier obtient la version épinglée, sans configuration supplémentaire, et ça disparaît dès que vous en sortez.

## Les nouvelles installations PHP sont préconfigurées

La première fois que Stack télécharge une version de PHP via vfox, il modifie une fois le `php.ini` de cette installation : un ensemble choisi d'extensions dont toute application non triviale a besoin mais que les builds officiels livrent désactivées (`pdo_mysql`, `pdo_pgsql`, `pdo_sqlite`, `sockets`, `sodium`, et quelques autres), OPcache activé, et des valeurs par défaut plus adaptées au développement (`date.timezone`, un `memory_limit` plus élevé, des limites d'upload plus larges). Autrement, vous tombez dessus une par une — un driver PDO manquant ici, un avertissement de fuseau horaire non défini là — exactement le genre de chose qui mange une après-midi sur une machine neuve.

## Vérifier un manifeste avant qu'il n'échoue à mi-chemin

Avant de transmettre un projet à quelqu'un d'autre, ou après avoir récupéré des changements qui touchent `stack.toml` :

```console
$ stack doctor --project
checking C:\Users\you\acme-api\stack.toml...
  language.php: OK (C:\Users\you\.vfox\cache\php\v-8.3.1\...\php.exe)
  service.mysql: OK (managed)
```

Cette commande valide les ports, les chemins des services, et tout `{PLACEHOLDER}` dans `[run].command` face à votre environnement réel — en chargeant d'abord le `.env` du projet, comme le fait `stack up` — sans démarrer le moindre processus. Si quelque chose cloche, vous obtenez la liste complète d'un coup, au lieu de la découvrir erreur par erreur, en plein milieu d'un `stack up`.

## Des commandes qui marchent depuis n'importe où

Pas besoin d'être dans `acme-api` pour rien de tout ça, une fois que Stack l'a vu au moins une fois :

```console
$ stack describe acme-api
$ stack restart acme-api
$ stack down acme-api
```

`stack describe` affiche tout ce que Stack sait d'un projet — chemins des binaires résolus, l'emplacement réel de `php.ini` (franchement difficile à retrouver à la main sous le cache versionné et haché de vfox), le fichier de log, le domaine routé — que le projet tourne ou non. `stack restart` l'arrête et le relance en une seule étape. Les deux résolvent le nom via un petit registre local de chaque projet que Stack a déjà vu, pas depuis votre dossier courant.

## Terminer la journée

```console
$ stack down --all
```

Arrête tous les projets et tous les services partagés d'un coup — Caddy compris. Un vrai 0 % de CPU, pas juste "le projet que je regardais à l'instant".

## Un mot rapide sur les domaines personnalisés

`acme-api.localhost` fonctionne sans aucune configuration parce que `.localhost` est réservé par la RFC 6761 — les navigateurs le résolvent vers la boucle locale sans fichier hosts. Si vous préférez `.test` (venant de Laragon ou Herd, c'est le suffixe le plus familier), ça demande une étape unique — [Custom Domains](https://sanayavo.com/stack/custom-domains.html) détaille la configuration exacte pour Acrylic DNS Proxy sur Windows.

## Pour aller plus loin

Voilà la boucle complète : créer, lancer, approuver une fois, obtenir un domaine routé en HTTPS, partager le manifeste, tout arrêter proprement. Tout le reste — la référence complète des champs de `stack.toml`, chaque option de la CLI, `[[clone]]` pour démarrer un projet à partir d'un simple manifeste — est dans la documentation :

- [Why stack?](https://sanayavo.com/stack/why.html) — l'argumentaire complet face à Docker/XAMPP pour ce problème précis
- [Manifest Reference](https://sanayavo.com/stack/manifest.html)
- [CLI Reference](https://sanayavo.com/stack/cli.html)
- Code : [github.com/sanayasfp/stack](https://github.com/sanayasfp/stack)

Je continue à utiliser Stack quotidiennement sur du travail client réel, donc ce qui est couvert ici, ce sont les parties sur lesquelles je m'appuie vraiment tous les jours — pas une démo choisie pour l'occasion.
