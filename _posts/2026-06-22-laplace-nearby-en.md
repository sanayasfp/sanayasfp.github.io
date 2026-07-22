---
layout: project
title: "Laplace Nearby: A WhatsApp Bot That Finds 'The Nearest Maquis' Without Torching the LLM Budget"
date: 2026-06-22 14:15:00 +0000
lang: en
tags: [System Design, Backend, AI/LLM, PostgreSQL, TypeScript]
type: project
github_repo: "https://github.com/sanayasfp/laplace-nearby"
---

In Abidjan, a huge number of local businesses — the neighborhood *maquis* (informal restaurant), the pharmacy, the guy who fixes phones — don't have a website, a Google Business listing, or even a formal street address half the time. What they do have is WhatsApp. **Laplace Nearby** starts from that observation: instead of building an app people have to download, build the search experience inside the app everyone already has open, and let people describe what they want in plain language — "I need a pharmacy," "j'ai envie de porcodjo" — rather than filling out a filter form.

The assistant itself is internally called **Simon**. This post is about how Simon is actually built — the parts I think are worth talking about, not a marketing pitch.

Repo: [github.com/sanayasfp/laplace-nearby](https://github.com/sanayasfp/laplace-nearby).

## Decision #1: the engine doesn't know it's talking to WhatsApp

The core of the system is `SimonEngine`, a small orchestrator that takes a channel-agnostic `InteractionRequest` and returns a channel-agnostic `InteractionResponse`. WhatsApp specifics — webhook payloads, message formatting, buttons and lists — live entirely in an adapter/renderer layer outside the engine. The engine itself just does four things on every message: load the user's session context, check rate limiting, hand the message to whichever conversational flow is currently active (idle, search, or register), and save the result — always emitting an analytics side effect with the flow name, the state transition, and the latency, whether or not anything interesting happened:

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

This isn't architecture for its own sake. It means the search logic, the registration flow, the rate limiting, and the metrics don't need to be rewritten if a second channel (SMS, a web widget, whatever) shows up later — they were never coupled to WhatsApp in the first place.

## Decision #2: don't call the LLM if you don't have to

Every incoming message needs an intent — is this a search, a registration request, small talk, a thank-you, an insult, "stop"? Routing every single message through an LLM call is the easy way to build this, and also the expensive and slow way. So intent classification is two-tiered.

`FastIntentDetector` runs first: a deliberately exhaustive set of regex patterns covering French, English, and local slang and abbreviations you'd actually see in a WhatsApp chat in Abidjan — "wesh," "cc," "gab" (cash machine), "essence/gazoil" (fuel), dozens of spelling variants of "merci," "stop," "annule." Its docstring says exactly what it's for: *"Reduce LLM costs and latency for unambiguous user requests."* Only when nothing matches does `IntentService` fall back to Gemini (`gemini-2.5-flash-lite`), with temperature 0, JSON response mode, and a tight 150-token cap, asking for structured output: the intent, an extracted search keyword, and a list of semantically related terms to widen the search ("j'ai envie de porcodjo" → keyword "porcodjo restaurant"; "mon habit est sale" → "pressing nettoyage vêtement").

Both paths report where the classification came from — `REGEX` or `LLM` — to a Prometheus counter (`simonIntentTotal`), so the fraction of traffic being deflected from the paid model is something you can actually watch on a dashboard, not something you have to guess at. And the Gemini call itself sits behind a circuit breaker: if it trips, the system doesn't crash the conversation, it degrades to a neutral "chitchat" response with confidence 0 and moves on.

## Decision #3: addresses in Abidjan don't work like addresses elsewhere

A huge share of real addresses given in chat aren't geocodable strings — they're descriptions: "je suis vers la cité Abdoulaye Diallo." `AddressCodingService` takes that kind of input, checks a semantic cache (exact hash first, then embedding similarity above a 0.88 threshold) to avoid re-paying for something already resolved, and if it's a miss, asks Gemini to turn it into a standardized address, an extracted neighborhood, and — importantly — a flag for whether the description is precise enough to geocode at all or whether the only honest answer is "ask the user to drop a GPS pin." There's even a dedicated dictionary of **Nouchi** (Abidjan street slang) terms feeding into this pipeline, because generic NLP tooling doesn't know what a "gbaka" stop or a given neighborhood nickname refers to.

## Decision #4: search is one SQL function, three signals, fused

This is the part of the codebase I'm most proud of. `search_nearby_places` is a single Postgres function that blends three independent ranking signals for every candidate business inside a radius:

- **Full-text rank** (`ts_rank_cd` against a French `tsquery`) — good when the user typed something close to the business's actual name or category.
- **Vector similarity rank** — cosine distance between the query embedding (Gemini `text-embedding-004`, `pgvector` with an HNSW index) and each place's embedding — good when the user described what they want in their own words instead of matching a label.
- **Geographic proximity** (`ST_Distance` over a PostGIS `geography` column) — because "closest" always matters, and can't be papered over by relevance alone.

These get combined with a weighted **Reciprocal Rank Fusion**: `score = w_fts/(k + fts_rank) + w_vec/(k + vec_rank) + w_prox * proximity_term`, with the weights shifting based on whether the user gave a keyword at all — 45/45/10 between text, vector, and proximity when there's a keyword to match; 85% vector-driven when there isn't, since full-text search has nothing to grab onto in a purely descriptive query. There's also a "premium tier" mechanism: a second ranking pass, partitioned by tier, guarantees paying/listed businesses a small quota of slots without letting them drown out relevance for the general result set.

I later went back and rewrote this same function for performance after noticing it was doing redundant work: merging two CTEs that were reading the same rows twice, adding a `LIMIT` inside the vector-ranking CTE specifically so the HNSW index can short-circuit instead of sorting the entire candidate pool, and replacing two separate full materializations (general results, premium results) with a single windowed pass using `ROW_NUMBER() OVER (PARTITION BY is_premium ...)`. That's a genuinely satisfying kind of fix — same output, measurably less work per query — and it's the sort of thing that only shows up once real usage patterns put pressure on a first draft.

## Decision #5: side effects are data, not actions

When a place gets registered, or an interaction needs logging for analytics, or a user needs a notification, the engine doesn't go do that work inline — it returns a plain `SideEffect` object describing what should happen. A `PgmqSideEffectDispatcher` collects these, groups them by target queue, and pushes them via `pgmq.send_batch` (Postgres's own message-queue extension) *inside the same database transaction* that marks a newly registered place as "queued." That detail matters: it means a place can't end up half-registered — visible in the app but never actually indexed for search — because the status update and the queue write either both commit or neither does. Supabase Edge Functions on the other end (an `analytics-worker`, a `place-embedding-worker`, a `place-tagging-worker`) drain those queues asynchronously.

## The stack, plainly

Fastify + TypeScript on Node.js, Prisma over Supabase Postgres (with `pgvector`, `pgmq`, PostGIS, `pg_cron`, and `pg_net` doing real work, not just sitting in a dependency list), Redis for caching and rate-limit state, Gemini for both NLU and embeddings, Geoapify for geocoding, and Prometheus metrics wired in from the start rather than bolted on later.

## Where it stands

`package.json` says `0.4.0-rc.1` and I mean to leave that context in: this is a real, working system I use to reason about hybrid search and conversational engineering, actively evolving toward a 1.0 — not a finished, scaled product with a client roster behind it. The channel-agnostic design exists specifically so that if this ever needs to be more than a WhatsApp bot, the engine underneath doesn't need to be rebuilt.

If you want to argue about the RRF weights, tell me PostGIS was overkill, or point out a better way to structure the side-effect dispatcher — the code's public: [github.com/sanayasfp/laplace-nearby](https://github.com/sanayasfp/laplace-nearby).
