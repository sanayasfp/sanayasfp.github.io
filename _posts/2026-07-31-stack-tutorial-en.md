---
layout: post
title: "The Complete Guide to Stack: Running Multiple PHP, Node, and Python Projects Without Docker"
date: 2026-07-31 09:00:00 +0000
lang: en
tags: [Rust, Tooling, PHP, System Design, Backend, CLI]
excerpt: "A real, hands-on walkthrough of Stack: scaffolding a PHP+MySQL project, understanding stack.toml, trust-on-first-use, automatic HTTPS, and the everyday commands, end to end."
---

I wrote a [short post]({% link _posts/2026-07-28-stack-dev-env-en.md %}) a few days ago introducing **Stack** — why I built it, and the core idea (native processes instead of containers). This one is different: it's the tutorial I wish existed when I first tried it. No pitch, just the actual workflow, command by command, on a real PHP + MySQL project, with every output shown exactly as Stack prints it today (v0.1.26).

If you just want the reference material, it lives at [sanayavo.com/stack](https://sanayavo.com/stack/) — this post is the guided tour through it.

## What you're actually installing

Stack is a single Rust binary. It doesn't run a daemon, doesn't need Docker Desktop, and doesn't touch your system PHP/Node/Python install. It delegates version management to two tools you'd likely install anyway — [vfox](https://github.com/version-fox/vfox) for PHP/Node, [uv](https://github.com/astral-sh/uv) for Python — and routes local domains through [Caddy](https://caddyserver.com). Stack's own job is gluing those three together around one file per project: `stack.toml`.

Windows is the only supported platform right now (PowerShell and cmd). macOS and Linux are planned, not shipped.

## Install and one-time setup

```powershell
irm https://github.com/sanayasfp/stack/releases/latest/download/stackenv-installer.ps1 | iex
```

Then, once, on this machine:

```powershell
stack setup
```

This does two things: it wires a hook into your PowerShell profile (so a project's pinned toolchain activates automatically every time you `cd` into it), and it installs vfox, uv, and Caddy at the versions Stack is tested against. It also runs `caddy trust` — more on why in a moment.

```console
$ stack setup
added the stack hook for pwsh
checking vfox/uv/caddy...
  vfox: OK (1.0.11)
  uv: OK (0.11.7)
  caddy: OK (2.11.4)
  caddy: local CA trusted (https://*.localhost works with no browser warning)
```

Restart your terminal once so the hook takes effect. You won't run `stack setup` again unless you switch machines.

## Your first project

Let's build something real: a small PHP API backed by MySQL. Two ways to start, depending on whether the project already exists.

### Starting from scratch

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

Language and service selection is a checkbox list — space to toggle, enter to confirm. Nothing is written until you confirm, so an accidental toggle just gets toggled back, not a reason to start the wizard over.

### Starting from an existing project

If `acme-api` already has a `composer.json` with `"php": "^8.3"` in it, skip the manual typing entirely:

```console
$ stack init
detected from existing project files:
  php 8.3 (from composer.json)
domain: acme-api.localhost
```

`stack init` reads `composer.json`, `package.json`, or `pyproject.toml` and pre-fills whatever version constraints it finds. Same wizard after that.

### Reading the manifest it wrote

```toml
[project]
name = "acme-api"
domain = "acme-api.localhost"

[language]
php = "8.3.1"

[service.mysql]
version = "8.0.35"
```

This file is meant to be committed. It's the whole "share a project" artifact — the non-container equivalent of a Dockerfile. Anyone who clones the repo and runs `stack up` gets the exact same PHP version and the exact same MySQL version, without installing either by hand.

## Telling it how to run

`stack.toml` doesn't know your dev-server command yet. Open it and add a `[run]` section:

```toml
[run]
command = "php -S 127.0.0.1:{port} -t public"
```

`{port}` is substituted with a real, allocated port at start time. If you omit `[run]` entirely and `[language.php]` is declared, Stack defaults to its own FastCGI engine (`php-cgi.exe` with real concurrent worker processes, fronted by a Caddy FastCGI route) instead of requiring you to spell out a command — the same "sensible default" pattern services already have. We'll come back to that.

## Bringing it up

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

Two things worth pausing on here, because they're not obvious from a screenshot.

### The confirmation prompt

`[run].command` and `[service.*].command` are literal program invocations — Stack runs them exactly as written, the same way `npm install` runs whatever's in `package.json`'s scripts. The first time a project's commands are seen, or the first time they change (say, after a `git pull` brings a different one), Stack prints them and asks you to confirm before anything actually executes. Decline and nothing spawns. Approve once and it's remembered in `~/.stack/trust.json` — every later `stack up` on this exact set of commands is silent. Pass `--yes` to skip the prompt in CI or a script.

### It's already on HTTPS

That `routed:` line only shows `http://`, but `https://acme-api.localhost` works right now too, with no browser warning. `stack setup`'s `caddy trust` step installed Caddy's own local CA into your system trust store — mechanically the same thing [mkcert](https://github.com/FiloSottile/mkcert) does. Nothing forces the redirect; both protocols are live on the same route.

## What "no container" actually buys you here

Open a second terminal, `cd` into a *different* project that also declares `mysql = "8.0.35"`, and run `stack up` there too:

```console
  service.mysql: already running, shared with other projects (pid 41232, port 3306)
```

One MySQL process, two projects, isolated by schema rather than by a separate container each. If you `cd acme-api` in a third terminal right now and just run `php -v` by hand — no `stack up`, no manifest command — you'll get 8.3.1, because activation happens ambiently on every prompt, not just inside `stack up`:

```console
$ cd acme-api
$ php -v
PHP 8.3.1 (cli) (built: ...)
$ cd ..
$ php -v
'php' is not recognized as an internal or external command
```

That's the actual point of Stack: `composer install`, a test runner, an IDE's own PHP interpreter setting — anything that shells out to `php` inside that folder gets the pinned version, with zero extra configuration, and it disappears the moment you leave.

## Fresh PHP installs are pre-configured

The first time Stack downloads a PHP version via vfox, it patches that install's `php.ini` once: a curated set of extensions any non-trivial app needs but official builds ship disabled (`pdo_mysql`, `pdo_pgsql`, `pdo_sqlite`, `sockets`, `sodium`, and a few others), OPcache turned on, and dev-friendlier defaults (`date.timezone`, a higher `memory_limit`, larger upload limits). You'd otherwise hit these one at a time — a missing PDO driver here, an unset timezone warning there — exactly the kind of thing that eats an afternoon on a fresh machine.

## Checking a manifest before it fails halfway through

Before handing a project to someone else, or after pulling changes that touched `stack.toml`:

```console
$ stack doctor --project
checking C:\Users\you\acme-api\stack.toml...
  language.php: OK (C:\Users\you\.vfox\cache\php\v-8.3.1\...\php.exe)
  service.mysql: OK (managed)
```

This validates ports, service paths, and any `{PLACEHOLDER}` in `[run].command` against your actual environment — loading the project's `.env` first, same as `stack up` does — without starting a single process. If something's wrong, you get the full list at once instead of discovering it one error at a time, mid-`stack up`.

## Commands that work from anywhere

You don't need to be inside `acme-api` for any of this, once Stack has seen it once:

```console
$ stack describe acme-api
$ stack restart acme-api
$ stack down acme-api
```

`stack describe` prints everything Stack knows about a project — resolved binary paths, `php.ini`'s actual location (genuinely hard to find by hand under vfox's version-hashed cache), the log file, the routed domain — whether the project is currently running or not. `stack restart` stops and starts it again in one step. Both resolve the name from a small local record of every project you've ever brought up, not from your current directory.

## Ending the day

```console
$ stack down --all
```

Stops every project and every shared service in one shot — Caddy included. Actual 0% CPU, not "the one project I happened to be looking at."

## A quick custom-domain aside

`acme-api.localhost` worked with zero setup because `.localhost` is RFC 6761-reserved — browsers resolve it to loopback without a hosts file. If you'd rather use `.test` (coming from Laragon or Herd, it's the more familiar suffix), that needs one one-time step — [Custom Domains](https://sanayavo.com/stack/custom-domains.html) covers the exact setup for Acrylic DNS Proxy on Windows.

## Where to go from here

That's the full loop: scaffold, run, trust once, get a routed HTTPS domain, share the manifest, tear it down cleanly. Everything else — the full `stack.toml` field reference, every CLI flag, `[[clone]]` for bootstrapping a project from just a manifest — is in the docs:

- [Why stack?](https://sanayavo.com/stack/why.html) — the fuller case against Docker/XAMPP for this specific problem
- [Manifest Reference](https://sanayavo.com/stack/manifest.html)
- [CLI Reference](https://sanayavo.com/stack/cli.html)
- Code: [github.com/sanayasfp/stack](https://github.com/sanayasfp/stack)

I'm still dogfooding this daily on real client work, so the parts covered here are the parts I actually lean on every day — not a curated demo.
