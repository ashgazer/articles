---
tags:
  - hcc-strategy
  - code-quality
  - agent-assisted-development
  - lightweight-approach/document-tags
---

# Navigating Core's Architectural Debt — Interactive Dashboard

*HCC · Expand HCC*

An interactive read of HCC's strategy for getting agents to produce higher-quality code in Core — and how each proposed pattern already maps onto Core's real ADRs and architecture tests.

Source: [Confluence — Our approach to navigating Core's architectural debt](https://gousto.atlassian.net/wiki/spaces/EH/pages/5639110718/Our+approach+to+navigating+Core+s+architectural+debt) · Lucy Lloyd

Sections: ① The problem · ② Constraints · ③ Patterns × Core · ④ The ADR process · ⑤ Doc → Core map

---

## ① The problem space

HCC is spending time enabling agents to **reliably produce higher-quality code**. Today, working with agents in Core is often either inefficient or actively detrimental.

> **The premise:** an agent learns the patterns it sees. In a codebase carrying entrenched architectural debt, that cuts the wrong way.

### 🧠 If the engineer *is* aware of the debt

They have to carry that knowledge in their steering loop with the agent — constantly correcting it, every prompt, every review. Slow and tiring.

### 🌀 If the engineer *isn't* aware

Working with the agent is more likely to lead to **pattern replication** and further degradation of production code. The debt compounds, silently.

### The flip we're aiming for

Make the architectural knowledge live *in the repo* — as guidelines, ADRs, and deterministic tests — so the agent picks it up automatically. Then pattern replication starts working **for** us instead of against us.

---

## ② Constraints of the approach

The strategy is deliberately shaped to be lightweight and to deliver value immediately — five guardrails keep it grounded.

### 📆 Helpful on day 1

No budget for big overhauls. Prioritise and sequence patterns that add value without large investment — value that compounds as adoption rises.

### 🧩 Adoptable in small increments

Patterns must be clear in isolation, low cognitive overhead — no deep theory, agent assistance available, and at least one HCC-built working example anchoring each one.

### 🤝 Efficient consensus

A lightweight way to agree on patterns and keep moving. Sketched as a lightweight ADR process (detail to follow).

### ✅ Deterministically verifiable

Per good harness engineering, adherence should be checkable by tooling wherever possible — architecture tests for pattern usage and a measurable adoption signal.

### 🛠️ Bigger interventions when warranted

Incremental is the default, not a hard rule. Where a sweeping, targeted refactor is the right move, we'll be equipped to do it.

---

## ③ Patterns being considered × where Core already stands

Five candidate patterns (not in priority order, not yet fully scoped). Each maps to its goal and how it maps to Core's existing ADRs / architecture tests today.

**Reading the status badges:**
- **Live in Core** — already an ADR + enforced test
- **In progress** — partially landed or being scoped
- **Proposed** — in the doc, no ADR yet

### 1. Structure Core's directories by domain — **Live in Core**

- **Goal:** Make cross-domain violations obvious at the point of editing.
- **Why it helps agents:** When a file's domain is visible from its path, an agent (or human) editing it can see immediately when a change reaches across a boundary it shouldn't.
- **Where Core stands — ADR 0002:** Domain code lives under one PSR-4 root: `src/app/<Domain>/` (namespace `App\<Domain>\`), with factories and tests mirroring it. No new shared trees.

### 2. Actions & rules named in domain language — **Proposed**

- **Goal:** Make business logic unit-testable, explicit, and easy to scan for duplication.
- **Why it helps agents:** Laravel-flavoured DDD: pull business logic out of fat controllers/models into named Actions and Rules. Explicit names make duplication visible and units trivially testable.
- **Where Core stands — No ADR yet:** Candidate for the next ADR. HCC would land one production example + an agent skill before squads adopt.

### 3. DTOs at HTTP & domain action boundaries — **Live in Core**

- **Goal:** Stop internal models leaking across boundaries; foundation for runtime API schema validation.
- **Why it helps agents:** DTOs at the edges decouple the wire/contract shape from internal Eloquent models, and give a place to enforce API schema validation at run time.
- **Where Core stands — ADR 0005:** Typed HTTP contracts — the realisation of this pattern in Core today.

### 4. Domain services accessed via interfaces — **Partial / In progress**

- **Goal:** Make domain entry points explicit and substitutable.
- **Why it helps agents:** Fronting a domain with an interface declares its public surface and makes it swappable in tests and across boundaries.
- **Where Core stands — Partly via ADR 0002 / 0004:** ADR 0002's "declared surface" + ADR 0004's PHPat boundary enforcement already push this way; not yet its own ADR.

### 5. Isolate Eloquent in a DB layer — **Proposed**

- **Goal:** Keep persistence coupling out of domain logic.
- **Why it helps agents:** Confining Eloquent to a dedicated DB layer stops ORM concerns bleeding into domain code, keeping the domain pure and testable.
- **Where Core stands — No ADR yet:** Candidate for a future ADR. Pairs naturally with the DTO and interface patterns.

### Summary table

| Pattern | Status | ADR / Where Core stands |
|---|---|---|
| Structure Core's directories by domain | **Live in Core** | ADR 0002 |
| Actions & rules named in domain language | **Proposed** | No ADR yet |
| DTOs at HTTP & domain action boundaries | **Live in Core** | ADR 0005 |
| Domain services accessed via interfaces | **Partial** | Partly via ADR 0002 / 0004 |
| Isolate Eloquent in a DB layer | **Proposed** | No ADR yet |

---

## ④ A draft ADR process

The first step: a lightweight, repo-native way to agree on patterns and commit them. Agreed with squads, but might look like this.

1. **HCC drafts the ADR as a PR** — Carrying the essential context — the problem the pattern addresses, the proposed approach, and the rationale.
2. **Announced in Slack** — So squads have visibility and a clear place to surface feedback, concerns, or objections. (Channel to be confirmed.)
3. **Discussion on the PR itself** — The conversation lives next to the decision, not in a separate doc or thread.
4. **Merge on reasonable consensus** — Once concerns are addressed and there's broad agreement, the ADR merges into the repo.
5. **HCC applies it in production first** — The pattern is used at least once in prod and any learning is captured in agent skills — so what lands is road-tested, not theoretical — before squads adopt.

### The twofold outcome

#### 🤝 Consensus on the pattern

Captured in the ADR itself — the decision lives next to the discussion that produced it.

#### 🤖 Context agents can pick up

Committed alongside the ADR, so when an agent hits that area of debt it has both the *awareness* and the *steps* to take.

### Why this is the load-bearing idea

The ADR isn't just a record for humans — it becomes part of the agent's working context. That's what makes adoption sustainable: the knowledge doesn't have to live in any one engineer's head.

---

## ⑤ The doc, mapped onto Core today

This strategy isn't theoretical for Core — the scaffolding already exists. `GUIDELINES.md` is the entry point, and `docs/adr/` holds the road-tested decisions. Here's the alignment.

| From the doc | In Core today | Status |
|---|---|---|
| Lightweight ADR process | `docs/adr/0001` — record architecture decisions (Status/Context/Decision/Consequences) | **Live** |
| Structure Core by domain | `docs/adr/0002` — domain code under one PSR-4 root `src/app/<Domain>/` | **Live** |
| Deterministically verifiable adherence | `docs/adr/0004` — PHPat rules running with PHPStan; `src/architecture/` | **Live** |
| DTOs at HTTP / domain boundaries | `docs/adr/0005` — typed HTTP contracts | **Live** |
| Contracts as first-class architecture | `GUIDELINES.md` — "Contracts are first-class" | **Live** |
| Actions/rules in domain language (Laravel DDD) | No ADR yet — candidate for the next one | **Proposed** |
| Isolate Eloquent in a DB layer | No ADR yet — candidate for the next one | **Proposed** |
| Domain services via interfaces | Partially implied by 0002's declared surface; not its own ADR | **Partial** |

### The short version

Core already has the **machinery** the doc describes — recorded decisions (0001), domain roots (0002), enforced architecture tests (0004), typed contracts (0005). The remaining doc patterns (domain-language actions, Eloquent isolation, interface-fronted services) are the **next ADRs to road-test and land**.

---

*Interactive summary built from the Confluence doc + Core's `GUIDELINES.md` and `docs/adr/`. Open the source page for the canonical version.*
