---
layout: project
title: "Project: Laplace Nearby - Architecture de Géolocalisation Temps Réel"
date: 2026-06-22 14:15:00 +0000
lang: fr
tags: [System Design, Backend, GeoSpatial, Go, Microservices]
type: project
---

## 1. Contexte & Executive Summary

**Entreprise / Client :** Développé pour un acteur majeur de la logistique urbaine et du transport du dernier kilomètre.  
**Projet :** **Laplace Nearby** est un moteur de géolocalisation et de calcul d'itinéraires en temps réel. Ce projet démontre ma capacité à concevoir des architectures backend capables de traiter des millions de requêtes spatiales avec une latence quasi nulle, tout en s'intégrant parfaitement dans un écosystème d'entreprise existant.

## 2. Le Problème Métier

Le client souffrait d'une infrastructure vieillissante incapable de gérer les pics de charge liés au suivi de milliers de livreurs en temps réel. Les requêtes pour trouver "le coursier le plus proche" prenaient plusieurs secondes, impactant directement les délais de livraison et l'expérience client.

**Objectif :** Refondre le service de géolocalisation pour garantir une latence sous les 50ms, tout en rendant l'architecture suffisamment modulaire pour être réutilisée par d'autres départements (marketing localisé, analyse de flotte).

## 3. Architecture & System Design

J'ai dirigé la conception et l'implémentation d'un système distribué et réactif :

- **Ingestion Temps Réel :** Utilisation de connexions **WebSockets** et de **gRPC** pour l'ingestion des flux GPS des coursiers, avec un découplage via un broker d'événements.
- **Indexation Spatiale In-Memory :** Remplacement des requêtes relationnelles lourdes par **Redis Geospatial (GeoHash)** pour les requêtes "nearby" (dans un rayon donné), offrant des temps de réponse de l'ordre de la milliseconde.
- **Source of Truth Permanente :** Stockage persistant et asynchrone des historiques de trajectoires dans **PostgreSQL avec l'extension PostGIS**, utilisé principalement pour l'analytique et l'audit.
- **Conception Modulaire :** Architecture Hexagonale pour isoler le domaine métier des détails d'infrastructure, facilitant les tests et l'ajout de nouveaux algorithmes de *matching*.

## 4. Défis Techniques & Solutions

- **Défi : Explosion du volume d'événements GPS ("Thundering Herd").**
  - *Solution :* Implémentation d'un mécanisme de *debouncing* (regroupement des mises à jour) côté client et d'un *rate-limiter* adaptatif sur l'API Gateway pour lisser la charge.

- **Défi : Complexité de l'indexation spatiale à très grande échelle.**
  - *Solution :* Partitionnement (Sharding) de la base Redis par zones géographiques (par villes ou grandes régions) pour distribuer la charge et éviter les goulots d'étranglement sur un seul nœud.

## 5. Technologies Utilisées

- **Backend :** Go (sélectionné pour ses performances exceptionnelles en concurrence avec les *Goroutines*)

- **Bases de données & Caches :** Redis, PostgreSQL, PostGIS
- **Communication :** gRPC, WebSockets, RabbitMQ
- **Infrastructure :** Conteneurisation (Docker), Orchestration (Kubernetes), Grafana/Prometheus (Monitoring)

## 6. Impact & Résultats

- **Performance :** Latence de matching (trouver le point le plus proche) divisée par 40, passant de **2 secondes à < 50ms**.

- **Coûts d'Infrastructure :** Réduction de **40% des coûts serveurs** grâce à l'efficacité du Go et au caching Redis, nécessitant moins de ressources que l'ancienne architecture Node.js.
- **Business :** L'amélioration de la fiabilité a directement contribué à une hausse de **15% du respect des délais de livraison (SLA)** lors du trimestre de lancement.
