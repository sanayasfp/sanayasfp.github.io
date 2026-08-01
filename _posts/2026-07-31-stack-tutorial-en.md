---
layout: post
title: "The Ultimate Guide to a Smooth Dev Environment"
date: 2026-07-31 09:00:00 +0000
lang: en
tags: [Rust, Tooling, PHP, System Design, Backend, CLI]
excerpt: "Tired of Docker melting your laptop? I spent a week perfecting Stack so you don't have to. Here is the ultimate step-by-step guide to a frictionless PHP + MySQL environment."
---

I [wrote about Stack]({% link _posts/2026-07-28-stack-dev-env-en.md %}) a few days ago — covering the short version of why I built it and the whole "no Docker" philosophy. But theory only gets you so far. If you're looking for the ultimate guide to actually achieving a buttery-smooth dev environment, this is it.

No elevator pitch. No marketing fluff. Just the raw, practical reality of getting a project running effortlessly, complete with every weird prompt and console output you'll encounter along the way.

If you just want the dry reference docs, you can find them at [sanayavo.com/stack](https://sanayavo.com/stack/). Otherwise, welcome to the guided tour of a better workflow.

## What you're actually getting

A smooth environment means zero bloat. Stack is just one Rust binary. There is no daemon hogging RAM in your system tray, no Docker Desktop spinning up your laptop fans, and absolutely nothing touching your global system PHP install.

Instead, it acts as highly intelligent duct tape. It leans on [vfox](https://github.com/version-fox/vfox) for managing PHP/Node versions, [uv](https://github.com/astral-sh/uv) for Python, and [Caddy](https://caddyserver.com) for instant local domains. Stack reads a simple `stack.toml` file in your project root and automatically wires these three tools together.

*Note: Stack is Windows only right now (PowerShell or cmd). Mac and Linux support are on the roadmap.*

## The Installation: Your one-time headache

A smooth day-to-day workflow requires a tiny bit of upfront setup. Run this to download the installer:

```powershell
irm https://github.com/sanayasfp/stack/releases/latest/download/stackenv-installer.ps1 | iex
```

Then, run this command *once* per machine:

```powershell
stack setup
```

This handles the heavy lifting behind the scenes. It adds a hook to your PowerShell profile (so the correct PHP or Node version magically activates when you `cd` into a directory), installs vfox, uv, and Caddy at heavily-tested versions, and runs `caddy trust`. That last step is crucial for local HTTPS — more on that in a minute.

```console
$ stack setup
added the stack hook for pwsh
checking vfox/uv/caddy...
  vfox: OK (1.0.11)
  uv: OK (0.11.7)
  caddy: OK (2.11.4)
  caddy: local CA trusted (https://*.localhost works with no browser warning)
```

**Restart your terminal now.** If you skip this, the PowerShell hook won't be active and you'll wonder why nothing is working. You will never have to run `stack setup` again unless you buy a new computer.

## Starting a project friction-free

Let's build something real to show off the workflow: a lightweight PHP API backed by MySQL.

### From scratch

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

It’s an interactive terminal checklist. Space to toggle, enter to confirm. It feels a bit different the first time, but nothing commits until you press enter. If you accidentally select Node instead of PHP, just hit space again. Zero stress, no do-overs required.

### If you already have a project

If your `acme-api` directory already has a `composer.json` file specifying `"php": "^8.3"`, Stack is smart enough to skip the manual typing:

```console
$ stack init
detected from existing project files:
  php 8.3 (from composer.json)
domain: acme-api.localhost
```

`stack init` automatically parses `composer.json`, `package.json`, or `pyproject.toml` and pre-fills the wizard for you.

### The magic artifact

Stack generates a highly readable manifest:

```toml
[project]
name = "acme-api"
domain = "acme-api.localhost"

[language]
php = "8.3.1"

[service.mysql]
version = "8.0.35"
```

Commit this `stack.toml` file to your repo. It is your ultimate "works on my machine" artifact—the lightweight, container-free equivalent of a Dockerfile. When a teammate clones the repo and types `stack up`, they instantly get the exact same PHP and MySQL versions without installing anything by hand.

## Teaching it how to run

Your `stack.toml` needs to know how to boot up your dev server. Open the file and add a `[run]` block:

```toml
[run]
command = "php -S 127.0.0.1:{port} -t public"
```

The `{port}` placeholder dynamically swaps for an available port at runtime. (Note: If you declare `[language.php]` but skip the `[run]` section, Stack falls back to its built-in FastCGI setup using `php-cgi.exe` and Caddy, which is incredibly solid. But for this guide, we'll keep it explicit.)

## `stack up` and watch it fly

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

There are two major things happening here that contribute to a seamless experience:

### 1. The security trust prompt

Because `[run].command` is a literal shell invocation, Stack runs it exactly as written. The first time Stack sees a new or modified command (like after a `git pull`), it asks you to confirm it. Say *yes*, and it remembers your choice in `~/.stack/trust.json`. Every subsequent `stack up` is blissfully silent. Need to bypass it for CI? Just pass `--yes`.

### 2. HTTPS works out of the box

The console says `http://`, but `https://acme-api.localhost` works instantly. No red browser warnings. Remember that `caddy trust` command from the setup phase? It installs a local Certificate Authority (similar to `mkcert`), meaning your local environment accurately mirrors production SSL from day one.

## What "no containers" actually means in practice

Open a second terminal, `cd` into a totally *different* project that also requires MySQL 8.0.35, and run `stack up`:

```console
  service.mysql: already running, shared with other projects (pid 41232, port 3306)
```

That’s the beauty of it. One MySQL process serving two projects, isolated by schema rather than burning your CPU and RAM on redundant containers.

But here is where the "smooth" factor really shines. Open a third terminal, `cd` into `acme-api`, and just run `php -v`. No `stack up`, no extra commands:

```console
$ cd acme-api
$ php -v
PHP 8.3.1 (cli) (built: ...)
$ cd ..
$ php -v
'php' is not recognized as an internal or external command
```

The version switch happens *ambiently* on every prompt thanks to that PowerShell hook. Your IDE, `composer install`, and your test runners all automatically get the correctly pinned version the second they enter the folder. Leave the directory, and it vanishes. No `source venv/bin/activate`, no `nvm use`, and no polluting your global system.

## Fresh PHP installs that don't suck

Usually, a fresh PHP installation means spending an afternoon hunting down "PDO driver not found" or "timezone not set" errors. Stack eliminates this entirely.

The first time it downloads a PHP version via vfox, it automatically patches `php.ini`. It turns on OPcache, bumps memory and upload limits, sets a timezone, and enables the extensions you actually need out of the box (`pdo_mysql`, `pdo_pgsql`, `pdo_sqlite`, `sockets`, `sodium`, etc.). It happens once, automatically, and you never have to think about it again.

## `stack doctor` — The ultimate pre-flight check

Before you hand off a project or start debugging a strange issue, run this:

```console
$ stack doctor --project
checking C:\Users\you\acme-api\stack.toml...
  language.php: OK (C:\Users\you\.vfox\cache\php\v-8.3.1\...\php.exe)
  service.mysql: OK (managed)
```

This command validates your ports, paths, and `{PLACEHOLDER}` values against your environment (loading `.env` first) without starting a single service. If something is broken, you get a clean list upfront instead of discovering cryptic errors halfway through startup.

## Command your setup from anywhere

Once Stack knows about a project, you don't even need to be in its folder to manage it:

```console
stack describe acme-api
stack restart acme-api
stack down acme-api
```

`stack describe` dumps everything—resolved binary paths, the exact location of your hashed `php.ini`, logs, and routed domains. Stack keeps a local registry of your projects, making global management effortless.

## Shutting it all down cleanly

When the day is over:

```console
stack down --all
```

Everything stops. Every project, every shared service, and Caddy. You get actual zero CPU usage, not a lingering phantom container you forgot to kill.

## A quick note on custom domains

Using `.localhost` guarantees a smooth experience because browsers automatically resolve it to the loopback address (RFC 6761) without you needing to hack your hosts file. However, if you are migrating from Laragon or Herd and absolutely need `.test` domains, there is a one-time setup using [Acrylic DNS Proxy](https://sanayavo.com/stack/custom-domains.html).

## Your New Workflow

Scaffold, run, trust once, enjoy automatic HTTPS, commit your manifest, and tear it down cleanly when you're done.

I use this exact workflow for real client work every single day. This isn't a theoretical happy path; it's a battle-tested blueprint for an ultimate, smooth dev environment.

For the complete `stack.toml` reference, CLI flags, and advanced features (like using `[[clone]]` to bootstrap from just a manifest), check out the official docs:

- [Why I built this](https://sanayavo.com/stack/why.html) — the longer rant against Docker/XAMPP for local dev
- [Getting Started](https://sanayavo.com/stack/getting-started.html)
- [Manifest Reference](https://sanayavo.com/stack/manifest.html)
- [CLI Reference](https://sanayavo.com/stack/cli.html)
- Code: [github.com/sanayasfp/stack](https://github.com/sanayasfp/stack)
