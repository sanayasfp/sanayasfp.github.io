---
layout: project
title: "Laplace Nearby : un bot WhatsApp qui trouve « le maquis le plus proche » sans exploser la facture LLM"
date: 2026-06-22 14:15:00 +0000
lang: fr
tags: [System Design, Backend, AI/LLM, PostgreSQL, TypeScript]
type: project
github_repo: "https://github.com/sanayasfp/laplace-nearby"
---

À Abidjan, une grande partie des commerces de proximité — le maquis du quartier, la pharmacie, le réparateur de téléphones — n'ont ni site web, ni fiche Google Business, ni même une adresse postale formelle la moitié du temps. Ce qu'ils ont, en revanche, c'est WhatsApp. **Laplace Nearby** part de ce constat : plutôt que de construire une application que les gens doivent télécharger, autant construire l'expérience de recherche dans l'application que tout le monde a déjà ouverte, et laisser les gens décrire ce qu'ils cherchent en langage naturel — « il me faut une pharmacie », « j'ai envie de porcodjo » — plutôt que de remplir un formulaire de filtres.

L'assistant lui-même s'appelle en interne **Simon**. Ce billet parle de la façon dont Simon est réellement construit — les parties que je trouve intéressantes à raconter, pas un argumentaire commercial.

Le dépôt : [github.com/sanayasfp/laplace-nearby](https://github.com/sanayasfp/laplace-nearby).

## Décision n°1 : le moteur ignore qu'il parle à WhatsApp

Le cœur du système est `SimonEngine`, un petit orchestrateur qui prend en entrée une `InteractionRequest` indépendante du canal et renvoie une `InteractionResponse` tout aussi indépendante. Les spécificités WhatsApp — format des webhooks, mise en forme des messages, boutons et listes — vivent entièrement dans une couche d'adaptateurs/renderers en dehors du moteur. Le moteur lui-même fait quatre choses à chaque message : charger le contexte de session de l'utilisateur, vérifier le rate limiting, transmettre le message au flow conversationnel actif (idle, recherche, ou enregistrement), puis sauvegarder le résultat — en émettant systématiquement un effet de bord analytique avec le nom du flow, la transition d'état et la latence, qu'il se soit passé quelque chose d'intéressant ou non :

```ts
async process(req: InteractionRequest): Promise<InteractionResponse> {
  const context = await this.contextManager.load(req.profileId);
  const flow = this.flowRegistry.get(context.session.activeFlow || IDLE);

  const limitResult = this.rateLimitService.check(context.rateLimit);
  if (!limitResult.allowed) {
    return { messages: limitResult.notify ? [Responses.tooManyMessages(...)] : [] , ... };
  }

  const response = await flow.handle(req, context, interactionId);
  await this.contextManager.save(req.profileId, response.contextUpdate);
  return { ...response, sideEffects: [summaryEffect, ...response.sideEffects] };
}
```

Ce n'est pas de l'architecture pour le plaisir. Cela signifie que la logique de recherche, le flow d'enregistrement, le rate limiting et les métriques n'auront pas besoin d'être réécrits si un second canal (SMS, un widget web, peu importe) arrive un jour — ils n'ont jamais été couplés à WhatsApp en premier lieu.

## Décision n°2 : ne pas appeler le LLM si on peut s'en passer

Chaque message entrant a besoin d'une intention — s'agit-il d'une recherche, d'une demande d'enregistrement, de bavardage, d'un remerciement, d'une insulte, d'un « stop » ? Faire passer chaque message par un appel LLM est la solution de facilité, et aussi la plus lente et la plus coûteuse. La classification d'intention est donc à deux étages.

`FastIntentDetector` s'exécute en premier : un ensemble volontairement exhaustif d'expressions régulières couvrant le français, l'anglais, et l'argot local qu'on croise réellement dans une conversation WhatsApp à Abidjan — « wesh », « cc », « gab » (guichet automatique), « essence/gazoil », des dizaines de variantes orthographiques de « merci », « stop », « annule ». Sa docstring dit exactement à quoi il sert : *« Reduce LLM costs and latency for unambiguous user requests »* (réduire les coûts et la latence liés au LLM pour les requêtes non ambiguës). Ce n'est que lorsque rien ne correspond que `IntentService` bascule sur Gemini (`gemini-2.5-flash-lite`), avec une température à 0, un mode de réponse JSON, et un plafond de 150 tokens, en demandant une sortie structurée : l'intention, un mot-clé de recherche extrait, et une liste de termes sémantiquement proches pour élargir la recherche (« j'ai envie de porcodjo » → mot-clé « porcodjo restaurant » ; « mon habit est sale » → « pressing nettoyage vêtement »).

Les deux chemins signalent leur origine — `REGEX` ou `LLM` — à un compteur Prometheus (`simonIntentTotal`), ce qui fait que la part du trafic détournée du modèle payant est quelque chose qu'on peut littéralement observer sur un dashboard, pas quelque chose qu'on devine. Et l'appel à Gemini lui-même est protégé par un circuit breaker : s'il se déclenche, le système ne plante pas la conversation, il dégrade vers une réponse neutre de type « bavardage » avec une confiance à 0, et continue.

## Décision n°3 : les adresses à Abidjan ne fonctionnent pas comme ailleurs

Une grande partie des adresses données dans le chat ne sont pas des chaînes géocodables — ce sont des descriptions : « je suis vers la cité Abdoulaye Diallo ». `AddressCodingService` prend ce type d'entrée, vérifie d'abord un cache sémantique (hash exact, puis similarité par embedding au-delà d'un seuil de 0,88) pour éviter de repayer un traitement déjà résolu, et en cas d'échec, demande à Gemini de la transformer en adresse standardisée, un quartier extrait, et — c'est le point important — un indicateur précisant si la description est assez précise pour être géocodée, ou si la seule réponse honnête est de demander à l'utilisateur d'envoyer un point GPS. Il existe même un dictionnaire dédié de termes **nouchi** (l'argot de rue abidjanais) qui alimente ce pipeline, parce qu'un outillage NLP générique ne sait pas ce que désigne un arrêt de « gbaka » ou le surnom d'un quartier.

## Décision n°4 : la recherche, c'est une seule fonction SQL, trois signaux, fusionnés

C'est la partie du code dont je suis le plus fier. `search_nearby_places` est une unique fonction Postgres qui combine trois signaux de classement indépendants pour chaque commerce candidat dans un rayon donné :

- **Le rang texte intégral** (`ts_rank_cd` sur un `tsquery` en français) — utile quand l'utilisateur a tapé quelque chose de proche du nom ou de la catégorie réelle du commerce.
- **Le rang de similarité vectorielle** — distance cosinus entre l'embedding de la requête (Gemini `text-embedding-004`, `pgvector` avec un index HNSW) et l'embedding de chaque lieu — utile quand l'utilisateur décrit ce qu'il veut avec ses propres mots plutôt qu'en reprenant une étiquette.
- **La proximité géographique** (`ST_Distance` sur une colonne `geography` PostGIS) — parce que « le plus proche » compte toujours, et ne peut pas être compensé par la seule pertinence.

Ces signaux sont combinés via une **Reciprocal Rank Fusion** pondérée : `score = w_fts/(k + fts_rank) + w_vec/(k + vec_rank) + w_prox * proximité`, avec des poids qui varient selon que l'utilisateur ait donné ou non un mot-clé — 45/45/10 entre texte, vecteur et proximité s'il y a un mot-clé à faire correspondre ; 85% piloté par le vecteur sinon, puisque la recherche texte intégral n'a rien à quoi s'accrocher dans une requête purement descriptive. Il existe aussi un mécanisme de « palier premium » : une seconde passe de classement, partitionnée par palier, garantit aux commerces payants/référencés un petit quota de places sans pour autant noyer la pertinence pour l'ensemble des résultats.

Je suis revenu plus tard réécrire cette même fonction pour la performance, après avoir remarqué qu'elle effectuait du travail redondant : fusionner deux CTE qui relisaient deux fois les mêmes lignes, ajouter une `LIMIT` à l'intérieur du CTE de classement vectoriel spécifiquement pour que l'index HNSW puisse s'arrêter court plutôt que de trier tout le pool de candidats, et remplacer deux matérialisations complètes séparées (résultats généraux, résultats premium) par une seule passe fenêtrée utilisant `ROW_NUMBER() OVER (PARTITION BY is_premium ...)`. C'est le genre de correction vraiment satisfaisant — même résultat, mesurablement moins de travail par requête — et c'est le genre de chose qui n'apparaît que lorsque l'usage réel met sous pression un premier jet.

## Décision n°5 : les effets de bord sont des données, pas des actions

Quand un lieu est enregistré, qu'une interaction doit être journalisée pour l'analytique, ou qu'un utilisateur doit recevoir une notification, le moteur n'exécute pas ce travail en ligne — il renvoie un simple objet `SideEffect` décrivant ce qui doit se produire. Un `PgmqSideEffectDispatcher` collecte ces effets, les regroupe par file cible, et les pousse via `pgmq.send_batch` (l'extension de file de messages propre à Postgres) *dans la même transaction de base de données* que celle qui marque un lieu nouvellement enregistré comme « en attente d'indexation ». Ce détail compte : cela signifie qu'un lieu ne peut pas se retrouver à moitié enregistré — visible dans l'application mais jamais réellement indexé pour la recherche — parce que la mise à jour du statut et l'écriture dans la file valident ensemble, ou aucune des deux. Des Supabase Edge Functions en bout de chaîne (un `analytics-worker`, un `place-embedding-worker`, un `place-tagging-worker`) vident ces files de manière asynchrone.

## La stack, sans détour

Fastify + TypeScript sur Node.js, Prisma au-dessus de Supabase Postgres (avec `pgvector`, `pgmq`, PostGIS, `pg_cron` et `pg_net` qui font un vrai travail, pas juste des lignes dans un fichier de dépendances), Redis pour le cache et l'état du rate limiting, Gemini à la fois pour le NLU et les embeddings, Geoapify pour le géocodage, et des métriques Prometheus intégrées dès le départ plutôt que rajoutées après coup.

## Où en est le projet

Le `package.json` indique `0.4.0-rc.1`, et je tiens à laisser ce contexte tel quel : c'est un système réel et fonctionnel qui me sert à raisonner sur la recherche hybride et l'ingénierie conversationnelle, en évolution active vers une version 1.0 — pas un produit fini, à grande échelle, avec un portefeuille de clients derrière. La conception indépendante du canal existe précisément pour que, si ce projet doit un jour dépasser le cadre d'un bot WhatsApp, le moteur sous-jacent n'ait pas besoin d'être reconstruit.

Si vous voulez débattre des poids de la RRF, me dire que PostGIS était superflu, ou pointer une meilleure façon de structurer le dispatcher d'effets de bord, le code est public : [github.com/sanayasfp/laplace-nearby](https://github.com/sanayasfp/laplace-nearby).
