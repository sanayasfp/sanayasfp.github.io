---
layout: project
title: "Project: Car Inspect AI - Fraud Detection & Validation System"
date: 2026-06-22 14:00:00 +0000
lang: fr
tags: [Computer Vision, System Design, Backend, Machine Learning]
type: project
company: "AssurAuto / Global Insurance Client"
role: "Senior AI / Backend Engineer"
live_demo: "https://example.com/car-inspect-demo"
github_repo: "https://github.com/sanayasfp/car-inspect-ai"
---

## 1. Executive Summary

**Car Inspect AI** est un système automatisé conçu pour valider l'état physique de véhicules et détecter les fraudes documentaires en s'appuyant sur des modèles avancés de **Computer Vision**. Ce projet illustre mon expertise dans la conception d'architectures Backend résilientes et le déploiement de pipelines d'Intelligence Artificielle en production.

## 2. Le Problème Métier (Problem Statement)

Dans l'industrie de l'assurance et de la location automobile, l'inspection des véhicules et l'authentification des documents sont des processus souvent manuels, lents et sujets à de nombreuses erreurs. Les fraudes (déclarations de dommages artificiels, faux documents) engendrent des pertes financières considérables.

**Objectif :** Concevoir une API capable d'automatiser l'analyse d'images de véhicules en temps réel afin de détecter instantanément les incohérences ou les fraudes, tout en garantissant une expérience utilisateur fluide.

## 3. Architecture & System Design

Afin d'assurer robustesse et scalabilité, j'ai opté pour une approche modulaire orientée microservices :

- **API Gateway & Routage :** Point d'entrée unique gérant l'authentification, le rate-limiting et la validation initiale des payloads.
- **Inférence Asynchrone (Event-Driven) :** Les images volumineuses sont déposées sur un Message Broker (Kafka/RabbitMQ) afin d'être traitées en arrière-plan par des *workers* ML spécialisés. Cela évite le blocage des requêtes HTTP.
- **Optimisation des Modèles :** Les modèles de détection (YOLO/ResNet) sont convertis et optimisés via **TensorRT** pour garantir une inférence rapide avec une faible empreinte mémoire, même sur des GPU contraints.
- **Stockage Hybride :** Conservation des images dans un Object Store (type AWS S3) et persistance des résultats analytiques dans une base relationnelle hautement disponible (PostgreSQL).

## 4. Défis Techniques & Solutions

- **Défi : Goulots d'étranglement lors des pics de charge.**
  - *Solution :* Mise en place de l'auto-scaling via Kubernetes sur les workers ML. Utilisation de la quantification (passage de FP32 à INT8/FP16) pour diviser par deux le temps d'inférence sans perte significative de précision.

- **Défi : Faux positifs liés à l'environnement (reflets vs égratignures).**
  - *Solution :* Implémentation d'un pipeline robuste de pré-traitement (Data Augmentation, normalisation de l'éclairage) et intégration d'une boucle *Human-in-the-loop* permettant aux experts de corriger les erreurs pour un ré-entraînement continu.

## 5. Technologies Utilisées (Tech Stack)

- **Backend :** Python (FastAPI), Go (pour les services nécessitant une très faible latence)

- **Machine Learning :** PyTorch, TensorRT, OpenCV
- **Infrastructure :** Docker, Kubernetes, Apache Kafka, PostgreSQL, S3

## 6. Impact & Résultats (Business Impact)

- **Performance :** Réduction du temps de validation d'une inspection de plusieurs minutes (manuel) à **moins de 800 millisecondes**.

- **Précision :** Taux de détection correcte de **94%** sur les anomalies critiques.
- **Scalabilité :** L'architecture a pu absorber une augmentation soudaine de **300%** du volume de requêtes de manière totalement transparente.
