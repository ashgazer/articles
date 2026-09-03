---
tags:
  - result-machine-learning
  - functional-testing
  - domain-scoping
  - architecture-tests
  - code-quality/result
---

# What the HCC team has been building

*Gousto · Core · HCC Team*

A platform-health initiative on the Core monolith: faster CI, a modern functional-test framework, clear domain boundaries, enforceable architecture, and an AI-agent-friendly codebase.

👋 Welcome aboard, **Ashik** — here's everything they shipped **16 Apr → 12 Jun 2026**, before your first Monday.

**Stats:**
- **~108** HCC commits
- **4** ADRs accepted
- **8** Domains relocated
- **7** Work themes
- **~2** Months

## The one-paragraph version

**HCC is the "healthy Core codebase" effort.** Core is Gousto's big Laravel monolith. The team has spent the last two months paying down structural and test-infra debt so that both humans *and AI agents* get fast, trustworthy feedback when they change it.

Four pillars: **(1)** make CI fast and the test suite single-engine (MySQL, no more SQLite); **(2)** replace the old StateDriver functional tests with a cloud-capable **"snapshot" framework** built on per-domain drivers and `/_test/` endpoints; **(3)** give every domain **one namespace and one CODEOWNERS block** (`App\<Domain>\`); **(4)** make architecture **enforceable** via PHPat arch-tests with graded feedback modes. All of this is wrapped in an **ADR process** and a set of **Claude Code skills** so the patterns stick.

## The work, by theme

Click a card to expand. Use the filters to focus. (Colours match the legend.)

Filter categories: All · 🧪 Functional-test framework · 📦 Domain scoping · 🏛️ Architecture tests · ⚡ CI & speed · ✨ Code quality · 🤖 AI / agent tooling · 🧹 Dead-code cleanup

### 🧪 Functional-test framework: The "snapshot" functional-test framework

A from-scratch rebuild of how functional tests set up state — closed-box, cloud-capable, and far less brittle than the old StateDriver.

- **Tickets:** HCC-44 · HCC-96 · HCC-119 · HCC-120 · HCC-106
- **PRs:** ~15 PRs
- **Span:** May–Jun

The flagship effort. Replaces the legacy StateDriver mechanism with a "snapshot" tester that talks to a running stack.

- **FunctionalSnapshotTestCase** with a `db:rebuild` reset between tests (HCC-96, HCC-106).
- A composable **ScenarioBuilder** for common setup (HCC-120).
- Made the framework **cloud-capable** — same tests run against real cloud or fake/LocalStack (HCC-44).
- Decomposed the tester surface so each domain plugs in its own driver (HCC-119).

### 🧪 Functional-test framework: Per-domain state "drivers"

Each domain got a typed driver + `/_test/` endpoints to inject state, instead of poking the DB directly.

- **Tickets:** HCC-97..103, 98, 99, 100, 101, 102
- **PRs:** ~10 PRs
- **Span:** May

Drivers added for: **ActiveSession, User, Menu, Recipe, DeliverySlot, BoxOptions, Postcode, Market, Order, DeliveryOption**.

- Generic `/_test/messages` event-dispatch endpoint (HCC-98).
- Test-only order-mutation `/_test` endpoints + OrderDriver methods (HCC-44).
- See the `write-state-driver` & `write-functional-test` skills for how to add one.

### 🧪 Functional-test framework: Migrating every test off StateDriver

Test-by-test migration to the snapshot style, then retiring the legacy mechanism entirely.

- **Tickets:** HCC-41/42/43/45/46/47/103
- **PRs:** ~12 PRs
- **Span:** May–Jun

Migrated: Messaging, Subscription, AccountExperience, Postcode, OrderPlacement, OrderPriceCalculation, OrderDeliveryMove, Observability, IESeeding, Promotion + cross-cutting tests.

- **HCC-47** then retired the legacy mechanism in blocks 0–8: removed dead code, dropped "snapshot/legacy" naming, rewrote the docs to the new model.
- LocalStack SQS/S3 reset between tests; SNS topics pre-created to silence noise (HCC-34, HCC-65).

### 📦 Domain scoping: One namespace per domain (ADR-0002)

Collapse competing folder schemes into a single `App\<Domain>\` root with one CODEOWNERS block each.

- **Tickets:** HCC-57 (POC: Box Ordering)
- **PRs:** ADR + rollout
- **Span:** May

Core had **four** competing ways to organise code (`app/<TypedFolder>`, `bounded-contexts/`, `packages/`, `components/`). ADR-0002 commits to one namespace per domain.

- Payoff: a nameable domain (`ls src/app/<Domain>/`), one greppable root, simpler CODEOWNERS, and **measurable coupling**.
- It's a **namespace rewrite, not a refactor** — internal shape travels as-is.

### 📦 Domain scoping: 8 domains physically relocated

Box Ordering, Box Delivery, Billing, Signup, Subscriptions, Menu Planning, Marketplace, Order Capacity Planning.

- **Tickets:** HCC-57/85/86/88/89/90/91/104
- **PRs:** 8 PRs
- **Span:** May–Jun

Each domain moved under `src/app/<Domain>/` with tests + factories mirroring the path, plus CI codeception splits wired per-domain.

- Box Ordering was the worked POC (HCC-57/85).
- CODEOWNERS audited for orphans against tracked files (HCC-108).

### 🏛️ Architecture tests: Architectural tests with PHPat (ADR-0004)

Enforce "this codebase's chosen patterns" with fast, deterministic, graded-mode rules.

- **Tickets:** HCC-105 · HCC-114 · HCC-115
- **PRs:** ADR + configs
- **Span:** May–Jun

Adopted **PHPat** (a PHPStan extension) so arch rules bolt onto existing tooling. Three feedback modes:

- **report-only** — visibility before a remedy exists (e.g. "measure unplaced classes", HCC-114).
- **baseline** — gate new violations, chip away at the pile.
- **no-baseline** — zero-tolerance on clean surfaces.
- Configs are split **by feedback mode**; promoting a rule = moving its `services:` entry (HCC-115).

### ⚡ CI & speed: CI speed-ups

Made the pipeline meaningfully faster: MySQL perf flags, tmpfs, bigger hosts, merged jobs, gen2 runners.

- **Tickets:** HCC-51 · HCC-33 · HCC-112 · HCC-34
- **PRs:** ~6 PRs
- **Span:** Apr–May

- MySQL perf flags + tmpfs + bigger functional host; tuned parallelism (HCC-51).
- Merged checkout + install-php-dependencies into one job (HCC-51).
- Fake-cloud functional tests run via a containerised toolchain (HCC-33).
- CircleCI managed jobs moved to gen2 resource classes (HCC-112).

### ⚡ CI & speed: Single-engine test suite — bye SQLite (ADR-0003)

Standardise on MySQL (the prod engine) and delete the entire SQLite compatibility layer.

- **Tickets:** HCC-80 (ADR) · HCC-81 (removal)
- **PRs:** 2 PRs
- **Span:** May

SQLite was kept for a "3–5 min" local loop but the divergence cost (dialect, type affinity, shims) stopped paying off.

- Removed SQLite CI jobs, `mysql2sqlite`, clone connections, schema-conversion command, and all `DB_ENGINE` branches.
- Roughly **halves** the DB-backed CI matrix. Faster MySQL runs are the follow-up investment.

### ✨ Code quality: PHPStan + CS-Fixer modernisation

Static analysis from nothing → level 1, wired into CI + pre-commit, plus a tightened CS-Fixer ruleset.

- **Tickets:** HCC-20 · HCC-35 · HCC-74
- **PRs:** ~10 PRs
- **Span:** Apr

- Reached full PHPStan **level 0**, then raised to **level 1** (HCC-20).
- Wired into CI with a result cache; runs in the **pre-commit hook** reusing a warm container.
- CS-Fixer moved `@PSR2 → @auto + @Symfony` and tightened to reduce diff churn (HCC-20, HCC-35).
- All `src/` files now included in analysis (HCC-74).

### 🤖 AI / agent tooling: Claude Code skills & agent tooling

Codify recurring workflows as skills so agents (and people) apply the team's patterns automatically.

- **Tickets:** HCC-17 · HCC-19 · HCC-32 · HCC-113 · HCC-114
- **PRs:** ~6 PRs
- **Span:** Apr–May

- **circle-ci-check** — watch CI & report build/test status (HCC-17).
- **tdd-workflow** — outside-in TDD guidance (HCC-19).
- **migrate-functional-test** & **apply-review-fixes** skills (HCC-113, HCC-114).
- Git **worktree** support for agents (HCC-32).

### 🤖 AI / agent tooling: ADR process scaffolding

Set up a lightweight ADR workflow so big decisions are recorded, reviewable, and linkable from guidelines.

- **Tickets:** HCC-56
- **PRs:** 1 PR
- **Span:** May

Established `docs/adr/` with a template, README, and the record-architecture-decisions ADR (0001). `GUIDELINES.md` is the entry point and links out to the relevant ADR per area.

### 🧹 Dead-code cleanup: Dead-code & config removal

Deleted code paths that no longer carry their weight, reducing surface area for everyone.

- **Tickets:** HCC-XX · HCC-121 · HCC-63
- **PRs:** several
- **Span:** May–Jun

- Removed dead customer-microservice config + partialAccounts code + customers DB connection.
- Removed orphaned test data and dead paracept machinery (HCC-121).
- FileBasedRepository hardening: `LOCK_EX` + strict false-check (HCC-64).

## Architecture Decision Records

The "why" behind the big moves. These live in `docs/adr/` — read the linked one when its area touches your task.

### ADR-0001 — Record architecture decisions

**Status:** Accepted

Establishes the ADR practice itself — we record significant architectural decisions as short markdown files in `docs/adr/`. The mechanism every other ADR below rides on.

### ADR-0002 — How to scope a domain in Core's code

**Date:** 8 May 2026
**Status:** Accepted

Every domain-bound file lives under **one namespace per domain**: `App\<Domain>\` for app code, with factories and tests mirroring the path, and functional tests under `functional-tests/tests/<Domain>/`.

**Why**
Core had three+ competing schemes, so "what is Box Ordering?" needed CODEOWNERS *and* tribal knowledge. That made coupling unmeasurable and ownership fragile.

**The deal**
A namespace rewrite, **not** a refactor — internal arrangement travels as-is. Clear single-team code moves first; coupled/shared/unattributed code waits for follow-ups. Box Ordering was the POC.

### ADR-0003 — Database engines used by the test suite

**Date:** 11 May 2026
**Status:** Accepted

Standardise the test suite on **MySQL** (the production engine) and remove the SQLite path entirely.

**Why**
SQLite was kept for a fast local loop, but the two engines diverge (dialect, type affinity, transactions, constraints) — green SQLite ≠ green MySQL — and the compat shims were a tax on every new model/query. The original trade no longer balanced.

**Trade-off**
Loses the advertised "3–5 min" loop until MySQL runs are sped up; halves the DB CI matrix. Removal done in bisectable slices (CI → config → app → docs).

### ADR-0004 — Architectural tests

**Date:** 20 May 2026
**Status:** Accepted

Adopt **PHPat** (a PHPStan extension) for architectural rules, so they reuse PHPStan's parser, baseline, reporter, and CI invocation — no second toolchain.

**Three feedback modes**
**report-only** (visibility, can't gate) · **baseline** (gate new, chip away) · **no-baseline** (gate everything on clean surfaces). A rule's mode = which config registers it.

**Keep it fast**
Scope subjects narrowly and split configs as rules accrete — static-analysis cost scales with subject-set size. Configs now split by feedback mode (update 2026-06-10).

## Rough timeline

How it unfolded — foundations first, then the big framework + relocation push.

**mid-Apr 2026 — Foundations: tooling & guardrails**
CircleCI skill, CircleCI MCP improvements, worktree support, tdd-workflow skill. The agent-friendly groundwork.

**late Apr 2026 — Static analysis lands**
PHPStan level 0 → 1, wired into CI + pre-commit with a result cache. CS-Fixer modernised. First CI speed-ups (MySQL flags, tmpfs).

**early May 2026 — ADR process + first decisions**
ADR scaffolding (HCC-56). ADR-0002 (domain scoping) and ADR-0003 (drop SQLite) proposed and accepted.

**mid-May 2026 — The big push begins**
SQLite removed. Box Delivery & Box Ordering relocated. Snapshot test framework + first drivers (Session, Menu, Recipe, DeliverySlot...).

**late May 2026 — Framework matures, domains move**
ADR-0004 (PHPat arch tests). Billing/Signup/Marketplace/Capacity relocated. Tests migrating off StateDriver in bulk. migrate-functional-test & apply-review-fixes skills.

**early-Jun 2026 — Cloud-capable + legacy retired**
Snapshot framework made cloud-capable (HCC-44). Legacy functional-test mechanism retired in blocks (HCC-47). Subscriptions/Menu Planning relocated. Arch rules split by mode.

## Glossary for day one

Terms you'll see in standup and PRs from the get-go.

- **Core** — Gousto's large Laravel monolith. The repo you're in (Gousto2-Core).
- **HCC** — The team/initiative behind this work — "healthy Core codebase". Tickets are HCC-NN.
- **Snapshot test** — The new functional-test style: closed-box, talks to a running stack, resets via `db:rebuild`, sets state through drivers.
- **Driver** — A typed helper (e.g. OrderDriver, MenuDriver) that injects test state via `/_test/` endpoints instead of touching the DB directly.
- **StateDriver (legacy)** — The old functional-test state mechanism — now fully retired (HCC-47).
- **/_test/ endpoints** — Test-only HTTP endpoints the drivers call to seed or mutate state in a running instance.
- **PHPat** — A PHPStan extension for writing architectural rules as first-class PHP. The basis of ADR-0004.
- **Feedback mode** — How strictly an arch rule is enforced: report-only / baseline / no-baseline. Set by which config registers the rule.
- **ADR** — Architecture Decision Record — a short markdown doc in `docs/adr/` capturing a significant decision + its why.
- **Domain scoping** — Putting every domain under one `App\<Domain>\` namespace + one CODEOWNERS block (ADR-0002).
- **CODEOWNERS** — GitHub file mapping paths → owning squads. ADR-0002 makes ownership a product of namespaces.
- **make dev-test** — How you run tests locally in this repo. e.g. `make dev-test FUNCTIONAL_TESTS="<filter>"`.

---

*Generated from `git log` + `docs/adr/` on this repo · Gousto2-Core · for Ashik's onboarding*
