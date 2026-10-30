---
layout: post
title: "The AI Illusion: Why feeding an LLM without data governance is a risk you can’t audit"
date: 2026-10-30 10:00:00 +0000
lang: en
tags: [AI/LLM, System Design, Data Governance, Security, Architecture]
excerpt: "Sending raw business data into an LLM with no controls isn’t innovation: it’s outsourcing your company’s strategic memory to a third party that will never bear the cost of your leaks. Here are the six invariants of robust governance."
image: /assets/the_ai_illusion/thumbnail_3x2_1200x800.png
---

A lawyer opens the company’s new AI assistant, uploads an exclusivity contract, and types: “Summarize the key obligations in five lines.” Ten seconds later she has a clean summary. It’s a small, real productivity win.

What she doesn’t see is what happened in those ten seconds. The application didn’t just send her question. It sent the whole contract: the counterparty, the prices, the termination clauses, the signatory’s name. All of it landed, as readable text, on a server run by someone else.

The same thing happens when an HR manager asks for a summary of a performance review, or a financial analyst digs into a client’s credit file. The question is short. The context that travels with it is enormous.

In nearly every executive committee, the directive for 2026 is the same: “We need AI, and we need it now.” Within weeks, API keys are created, language models are wired into internal databases, and whole workflows are handed to third-party algorithms. The demos are impressive and the productivity gains are real.

This article is not an argument against that. It’s an argument that the companies that get data governance right will deploy AI faster and in more places, because the foundation underneath is safe. The companies that skip it will eventually face one simple question they cannot answer. We’ll come back to it at the end.

## 1. The Problem: Every Request Carries Your Data

Even when your contract with the AI vendor forbids training on your data, and even when the connection is fully encrypted in transit, the raw text still arrives in readable form at a third party. Contracts limit what the vendor is allowed to do with it. They don’t change what is exposed if that vendor suffers a breach, if a log is misconfigured, or if a legal request lands on their desk.

The risk grows with multi-agent architectures, where a router agent delegates work to specialized sub-agents that share a common memory. A single name or revenue figure injected into that shared state is now visible to every agent in the chain, and to every log, cache and tracing tool they write to.

![A router agent writes raw data into a shared state, later read by three sub-agents](/assets/the_ai_illusion/fig1_shared_memory.png)
*Figure 1: Injection of raw data into a multi-agent shared memory.*

Sending raw business data into an LLM with no controls isn’t innovation. It’s outsourcing your company’s strategic memory to a party that will never bear the cost of your leaks.

## 2. Why Naive Anonymization Fails

Some teams, aware of the danger, put a basic filter in front of the model: generic Named Entity Recognition (NER) tools, or regular expressions that redact proper nouns. On paper, that sounds reasonable. In production it tends to fail in three ways, and the real danger is that it gives leadership a sense of safety that stops them from asking questions.

### Failure 1: The filter doesn’t understand your business
Generic detection tools are trained on everyday text. They confuse company names with common words (“Services”, “Transit”, “Holding”), and they often miss a company whose name contains such a word, like “NET-POLY SERVICES”. Internal identifiers such as tax codes or registration numbers don’t match standard formats, so they slip through.

### Failure 2: Hiding the name isn’t hiding the company
Consider this example:
* **Raw text:** “Company X posted 4.2 billion FCFA in revenue in Abidjan in 2024.”
* **After a naive filter:** “COMPANY_1 posted 4.2 billion FCFA in revenue in Abidjan in 2024.”

The name is gone, but the exact revenue and the city are enough to identify the company with a single search. A modern model, with its vast training knowledge, can often do the same in a fraction of a second.

### Failure 3: A prompt is a request, not a control
An instruction like “Never repeat client names” is a probabilistic nudge. The model will usually comply, until it doesn’t. There is a subtler problem too. Depending on the vendor and the settings, some models return their intermediate reasoning in a channel separate from the final answer, a kind of scratchpad handed back to you. If your protection only inspects the final answer, raw data can travel through the scratchpad unchecked.

Believing a system prompt that says “be discreet” protects your data is like locking an armored door and leaving the key in the lock outside.

## 3. The Six Invariants of Robust Governance

If instructions and basic filters aren’t enough, what is? Most architects sketch a linear pipeline: data comes in, gets filtered, goes to the model, comes back. A robust system looks different. It is defense in depth, where every boundary carries its own protection, independently of the others.

What follows are six invariants. They are contracts that any architect can check against their own system, whatever the stack, the model vendor or the team size.

![The six governance invariants, from the deterministic and model paths through to rehydration](/assets/the_ai_illusion/fig2_six_invariants.png)
*Figure 2: The six governance invariants applied to LLM workflows.*

### Invariant 1: Minimize at the source
What isn’t in the returned data can’t leak. The tools exposed to AI should never be the same ones exposed to your human developers. Pagination is enforced, volumes are capped, and unnecessary fields are removed entirely. The API doesn’t return a full customer record just to compute a ratio. It returns the three numbers actually needed: no contacts, no addresses, no names.

This isn’t an AI-specific rule. It’s a data-exposure rule that AI simply makes urgent.

### Invariant 2: Never let a model resolve what a deterministic path can decide
A ranking, a ratio lookup or a simple extraction requires no judgment, so it has no business going through your most expensive and least predictable model. The routing can be a business rule, a choice in the interface or a lightweight classifier. The principle doesn’t prescribe how.

Two honest caveats. First, if the router is itself a model, it is still a model: keep it small, tightly constrained, and feed it only protected text. The goal is to shrink exposure, not to claim it disappears. Second, this invariant has a benefit that often convinces a board before security does: it’s cheaper and faster.

A fun example comes from Anthropic itself. In March 2026, Claude Code’s source code was accidentally leaked through a source map published on npm. It showed that the tool detects user frustration, meaning whether someone is swearing at it, with a plain regular expression, not a model[^1]. A regex is instant and free. An inference call is neither, for a question that needs no judgment at all.

### Invariant 3: Treat every text-to-model boundary as a mandatory, reversible protection point
Any data crossing the boundary between your system and a language model, inbound or outbound, should be protected by a reversible mechanism such as substitution, pseudonymization or tokenization. The common mistake is protecting only the obvious response field. Check every channel the vendor exposes, including the reasoning channel described in section 2.

One caution: pseudonymized data can still count as personal data under regulations like GDPR. This invariant reduces your exposure. It doesn’t remove your legal obligations.

### Invariant 4: Rehydrate at the last legitimate moment
Substitution should be reversed only at the very last step, right before the answer is shown to a human who is authorized to see the real values. The model works blind from start to finish. Only the human gets the answer in the clear, and only at the end of the chain.

### Invariant 5: A deterministic guardrail has the final say on plausibility
The first four invariants protect confidentiality. This one protects correctness. Any response coming out of a deterministic path must pass through a non-probabilistic check, such as a threshold or a comparison with a known reference, before it goes out. If the result looks aberrant, the system falls back to a slower, supervised path rather than letting an unverified answer through.

### Invariant 6: Every agent-to-agent hop is its own trust boundary
This is the invariant most often forgotten, because it doesn’t show up in a static layered diagram. An agent’s output isn’t automatically safe for the agent that called it.

Here is the scenario. A sub-agent, believing it is “cleaning up its own output” before replying, de-anonymizes data that was meant to stay protected for the agent that delegated to it. The sub-agent did exactly what it was told. It just didn’t know its answer was headed for another link in the chain, one that was supposed to keep reasoning over pseudonyms.

There is an aggravating corollary. When conversation history crosses several agents, turns from different agents end up mixed in one context. Without explicit provenance tagging, you can no longer tell which trust boundary applies to which piece of text.

![Before/after comparison of a delegation between two agents, showing where the rehydration point moves](/assets/the_ai_illusion/fig3_agent_hop.png)
*Figure 3: Agent-to-agent trust boundaries and rehydration placement.*

## 4. For Engineers: How We Implement the Invariants

This is one possible implementation among many. The tools are specific to our stack. If your system looks nothing like ours, the invariants still hold; only their embodiment changes. Non-technical readers can skip to section 5.

1. **Anonymization gate (invariant 3):** Before anything is classified, the user’s free text passes through a strict gate that combines NLP libraries with deterministic substitutions. Names and identifiers become unique tokens (for example `CORP_A83B`). The correspondence table goes straight into an encrypted vault: HMAC-SHA256 for the token, Fernet encryption for the real value at rest.
2. **Router (invariant 2):** The pseudonymized question goes to a deliberately small, fast, low-cost LLM whose only job is to classify intent. Ranking, ratio lookup or simple extraction takes the **fast path**: a tool from a bounded allowlist is called directly, and the heavy model is never invoked. Risk analysis or interpretation takes the **slow path** and goes to the main agent.
3. **Minimized tools (invariant 1):** On both paths, internal data is reached through tools exposed via the Model Context Protocol (MCP), cleaned at the source and never the same tools as those given to human developers.
4. **Slow path (invariant 3, outbound):** Tool results are pseudonymized before they enter the heavy model’s context, so it reasons and writes entirely over pseudonyms.
5. **Fast path (invariant 5):** A plausibility guardrail, a simple ratio compared against a reference value, blocks any aberrant automatic answer and forces a fallback to the slow path for review.
6. **Rehydration (invariant 4):** Only once the response is generated and formatted does the system query the vault and swap tokens back for real values, right before display to the authorized user.

![Fast-path and slow-path routing between direct tool calls and the heavy-model main agent](/assets/the_ai_illusion/fig4_fast_slow_path.png)
*Figure 4: Fast-path / slow-path routing architecture.*

> **Observed results:** *(Add here your real production metrics — e.g., 72% of queries served via the fast path without heavy model invocation, 4x latency reduction, and 65% API cost reduction).*

## 5. Verify, Don’t Assume

A defense that has never been verified under real conditions is a defense you only hope exists.

Here is a scenario that is far from theoretical. A control meant to transform every message before it reached the model was written, reviewed and merged. The tests passed. For weeks, the team believed the protection was active. It wasn’t. An assumption about the exact shape of the incoming data was wrong, so the control ran against the wrong structure, silently, with no error and nothing to raise an alarm.

A test that passes without actually going through the protection code proves nothing. A code review is not proof of execution, and a clean architecture diagram is not proof that the path it draws is the path taken at runtime. The only proof that counts is a test replayed against the real system: the real model, the real data, the real call chain.

Every claim of compliance, such as “this is anonymized” or “this data never leaves in the clear,” should come with replayable proof, available on demand. Otherwise it isn’t a guarantee. It’s a hypothesis that hasn’t been disproved yet.

![A control that branches on the content shape of a received message](/assets/the_ai_illusion/fig5_wrong_shape.png)
*Figure 5: The control illusion — silent failure when data format mismatches expectations.*

## 6. The Audit Trail

Governance doesn’t stop at real-time protection. It requires traceability. When a banking regulator, a GDPR auditor or a major client reviews how AI is used in your organization, they won’t accept statements of intent. They will ask for evidence.

A responsible enterprise AI system keeps a tamper-evident, append-only audit log, kept separate from ordinary conversation logs and chat history. For every event in an agent’s lifecycle, it records a pair:
* the raw data as originally received,
* the data actually transmitted to external models.

The rule that matters most is that **no configuration flag or single administrator can turn this log off.** During load spikes, a team’s first instinct is often to disable logging “for performance”. If that switch exists, then on the day a leak happens you won’t be able to prove its scope or identify what was exposed.

![A request passing through four controls, three of which can be bypassed by a flag and one that can't](/assets/the_ai_illusion/fig6_four_controls.png)
*Figure 6: Why the audit log must be tamper-proof and impossible to bypass via configuration flags.*

There is a catch, and it deserves to be stated plainly: this log is itself one of the most sensitive datasets in the company, because it contains the raw data you worked so hard to protect. Treat it accordingly. Encrypt it, restrict and separate access to it, define retention limits, and work out with your legal counsel how it coexists with erasure requests under regulations like GDPR.

Finally, add retroactive leak scanning: background processes that keep re-scanning archived audit history for sensitive patterns that slipped past the first filters. When one is found, the scanner never edits the original log. It alerts the security team, who can tighten the rules before a small gap becomes an incident.

![The audit pipeline, from event to immutable log to asynchronous scan to alert](/assets/the_ai_illusion/fig7_audit_pipeline.png)
*Figure 7: Immutable audit pipeline and asynchronous leak scanning.*

## 7. Conclusion: AI Is a Data Engineering Problem

“Garbage in, garbage out” has a modern cousin: secret in, exposure out. AI isn’t a cosmetic layer you bolt onto a corner of your information system to satisfy the board. It changes how information flows, how it is segmented, how it is anonymized and how it is archived.

The companies that win with AI won’t be the ones that deployed the most agents the fastest. They will be the ones that built a safe foundation first. A good brake system is what lets a car drive fast. With these controls in place, teams can say yes to more use cases, with less debate and fewer late-stage vetoes.

So before you sign the next AI contract, or celebrate the rollout of your latest agent, gather your architects and ask one question:

> **“If an external auditor sent by our biggest client showed up tomorrow at 8am, could you prove to me, line of code by line of code, that not a single name, identifier or confidential figure left our network in the clear over the past month?”**

If the answer is silence, or “our vendor’s clauses”, or “a prompt that tells the AI to be careful,” you have your next project.

---

### The Governance Checklist: 8 boxes to check

Before you leave the room, eight boxes to check (the six invariants, plus the testing and auditing essentials):

- [ ] **Do you minimize at the source?** (Invariant 1)
- [ ] **Do you keep the expensive model off simple requests?** (Invariant 2)
- [ ] **Is every text-to-model boundary protected by a reversible mechanism?** (Invariant 3)
- [ ] **Do you rehydrate only at the last legitimate moment?** (Invariant 4)
- [ ] **Does a deterministic guardrail have the final say on plausibility?** (Invariant 5)
- [ ] **Is every agent-to-agent hop its own trust boundary?** (Invariant 6)
- [ ] **Is every control verified under real conditions, not just assumed?** (Section 5)
- [ ] **Is the audit log impossible to disable by flag or by a single administrator?** (Section 6)

---

[^1]: Alex Kim, “The Claude Code Source Leak: fake tools, frustration regexes, undercover mode, and more”, March 31, 2026. The leak came from a source map file published by mistake on npm. The file `userPromptKeywords.ts` contains the regex used for frustration detection.
