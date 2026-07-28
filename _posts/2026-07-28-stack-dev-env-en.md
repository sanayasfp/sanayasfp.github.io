---
layout: project
title: "Stack: A Native, Zero-Container Dev Environment Manager"
date: 2026-07-28 12:00:00 +0000
lang: en
tags: [Rust, Tooling, System Design, Backend, CLI]
type: project
github_repo: "https://github.com/sanayasfp/stack"
---

When you are maintaining a legacy application while building its modern replacement, your local development environment quickly becomes a battlefield. At work, we had a legacy app running on an old version of PHP, Vue 2 (requiring an outdated Node version), and Python 3.7. Alongside it, we were bootstrapping the new architecture with PHP 8.4, Vue 3, and Python 3.12. 

I personally managed this with a bespoke combination of Laravel Herd for PHP, `uv` for Python, and `nvm` for Node. It worked for me because I took the time to configure it. But for our junior developers and fresh interns, asking them to juggle these tools just to run both projects simultaneously was an absolute nightmare. They were fighting path conflicts and environment variables instead of writing code. That's why I built **Stack**: a native, zero-container multi-project dev environment manager that makes switching contexts as simple as typing `stack up`.

Code's here: [github.com/sanayasfp/stack](https://github.com/sanayasfp/stack).

![Stack Environment Manager Screenshot](/assets/stack-screenshot.png)

## The real constraint: Onboarding and Cognitive Load

Before touching the architecture, the real constraint on a project like this is developer experience. Interns shouldn't need to learn Docker network bridges or configure five different version managers to fix a bug in the legacy API. The goal was simple: they should be able to clone the repo, run a single command, and get a working local domain routed to the right processes, regardless of whether the project uses PHP 5.6 or 8.4.

## Why not just use Docker?

Containers solve the "works on my machine" problem by shipping an entire OS layer per project. But that comes with virtualization overhead and idle daemons eating RAM. 

Stack solves the same problem differently. It pins exact versions per project through version managers you'd install anyway (delegating to tools like `vfox` and `uv`), shares one downloaded binary across every project needing that version, and runs everything as plain child processes. No hypervisor, no Docker Desktop taking up 4GB of RAM for a simple API. Every project pinning the same service and version shares one running instance, isolated by schema.

## Under the hood: Simplicity over Magic

A `stack.toml` file in your project defines exactly what is needed:

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

When you run `stack up`, the orchestrator reads the manifest, wires up the local domain (e.g. `acme-api.localhost`), starts MySQL if it's not already running, boots the PHP dev server, and reverse-proxies the traffic. All native, all blazing fast. It doesn't try to reinvent package management; it delegates the heavy lifting of language downloads to `vfox` and `uv`, and the routing to Caddy.

## Where it actually stands right now

I want to be precise about this: I started coding Stack in Rust just last week and pushed the first release two days ago. I have completely uninstalled Herd, Laragon, and my previous manager scripts. I am now exclusively dogfooding Stack for my daily work — using it to prove it works, find edge-case bugs, and add new features to improve it. 

It currently supports Windows (PowerShell & cmd), which was the immediate need, with macOS and Linux on the roadmap. It's not just a weekend experiment anymore; it's becoming the foundational tool that keeps our team's local development sane.

You can grab the installer from the repo or read the official documentation:
- Repo: [github.com/sanayasfp/stack](https://github.com/sanayasfp/stack)
- Docs: [sanayavo.com/stack/](https://sanayavo.com/stack/)
