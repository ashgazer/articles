---
tags:
  - business-logic
  - structural-boundary
  - core-service-layer
  - dependency-rules
---

# 8. A structural boundary for Core's service layer

Date: 2026-09-02

## Status

Proposed

## Context

Business logic for a use case has no fixed address today. It lives wherever
the call site happened to need it: a controller action, a job's `handle()`,
an event listener, a console command, or — because an Eloquent model is the
one thing every layer can reach — the model itself, via an accessor, a local
scope, or an observer.

Nothing marks where a given piece of behaviour is *supposed* to live, and
nothing stops a new one landing wherever's convenient. There's no compiler
error, no test failure, no lint warning for putting a business rule in a
controller — only a convention, enforced by whoever happens to review the
PR.

## Problem

A rule with no fixed address gets reimplemented at whatever call site needs
it next, and the copies drift — each one encoding a slightly different
version of "is this order cancellable" or "can this user place an order,"
disagreeing at the edges in ways nobody notices until a bug report ties two
of them together.

None of that logic can be unit tested without a database, because it's
entangled with the thing that talks to one — an accessor or a controller
method can't be exercised without booting the framework and seeding rows.
Coverage for it only exists at the acceptance level, where the HTTP or job
round trip happens to be the one seam that can reach it.

And because there's no structural boundary, extraction work doesn't stick:
today's PR can move a rule out of a controller into a clean class, and next
month's PR adds a new one straight back into a different controller,
because nothing detects or blocks that. Without an enforceable line, this is
a one-off cleanup repeated forever, not a property of the codebase.

## Decision

This ADR introduces a structural boundary around Core's service layer —
enforced by two directional dependency rules, not by review convention.

Define **Core's service layer** as the one legal address for business
logic: two namespaces per domain, `App\<Domain>\Actions\*` (orchestrates a
single use case) and `App\<Domain>\Rules\*` (a named, composable business
predicate or policy), named in the domain's own language.

The service layer is a safe space, policed from both directions so the
boundary is structural rather than a naming convention:

**1. Outbound — what an Action or Rule may depend on.**
A class in `Actions\*` or `Rules\*` may depend only on:

- its own domain's `Actions\*`, `Rules\*`, `Data\*`
- another domain's published `Interfaces\*` port ([ADR
  0009](0009-domain-services-accessed-via-interfaces.md)) — never that
  domain's concrete Action, Rule, or Model directly
- a fixed, named list of framework primitives (PSR interfaces, `Carbon`,
  `Illuminate\Support\*` helpers) — extending this list is a deliberate,
  reviewed edit to the rule, not an unchecked import
- *(temporary, until [ADR 0010](0010-where-db-models-may-be-used.md)
  lands)* an Eloquent model, inside the class's own body only — the
  interior isn't clean yet, only the boundary this ADR draws is

It may never accept an Eloquent model in a **public method signature** —
that narrower wedge is what 0010 drives all the way through.

**2. Inbound — who may depend on the service layer, and on what else.**
A public entry point (controller action, job `handle()`, listener, console
command, observer, `FormRequest`/validation rule) may depend only on:

- its own domain's `Actions\*`, `Rules\*`, `Data\*` — the whole point being
  that it resolves inputs and calls exactly one Action, with no conditional
  of its own on business state
- the same fixed, named list of framework primitives (`Illuminate\Http\*`,
  PSR interfaces)

It may **not** depend on an Eloquent model, a repository, or another domain
directly — regardless of whether a specific leak surface for that
dependency has been named elsewhere. Stating this as an allow-list rather
than a deny-list is what turns "no conditional on business state" from a
review-only judgement into a structural property: a class that cannot
import the model cannot branch on it, and a listener that cannot import a
repository cannot grow a query. It also means we don't have to anticipate
every leak surface in advance — anything not on the list is blocked by
construction.

## Consequences

Logic becomes greppable — one class per rule — and unit-testable without a
database, since a Rule or Action no longer has to be reached through a
framework boot to be exercised. Entry points thin out, which shrinks the
surface [ADR 0010](0010-where-db-models-may-be-used.md) later has to
isolate.

This only cleans the boundary, not the interior — an Action's own body can
still build an Eloquent query internally (allowed above, temporarily), so
full decoupling from persistence still depends on [ADR
0009](0009-domain-services-accessed-via-interfaces.md) and [ADR
0010](0010-where-db-models-may-be-used.md). Two styles (old call sites vs.
new Actions) will coexist until burndown; needs the same baseline/no-baseline
ratchet as [ADR 0004](0004-architectural-tests.md) so "which regime am I in"
stays answerable by grep, not by asking around.

The allow-list is a stronger guarantee than a deny-list — it blocks leak
surfaces we didn't think to name, not just the ones we did — but it costs
more to keep current: every legitimate new dependency (a logger, an auth
facade) has to be added to the allowed set or the build breaks for reasons
that have nothing to do with this ADR. Wildcarding by namespace (`Rules\*`,
not each Rule class by name) keeps that cost to "add a namespace
occasionally" rather than "add a class every time."

## Possible enforcement mechanisms

Using [ADR 0004](0004-architectural-tests.md)'s PHPat setup and its three
feedback modes, this decision is two rules — one per direction above — each
run as its own baseline so existing violations are grandfathered rather than
blocking:

- **Outbound rule:** `Actions\*`/`Rules\*` classes must not depend on
  anything outside the allow-list above. These are new classes, so almost
  nothing violates this on day one — it can plausibly start at no-baseline
  and only need a baseline once the temporary Eloquent-in-body allowance is
  exercised.
- **Inbound rule, baseline and burn down:** entry points (Controllers, Jobs,
  Listeners, Observers, `FormRequest`s) must not depend on anything outside
  {`Actions\*`, `Rules\*`, `Data\*`, named framework primitives}. Almost
  every existing controller fails this today (they hold models via
  route-model binding), so it starts baselined; each PR that migrates a
  controller to the allow-list shrinks the baseline by one entry. This is
  what actually makes "no conditional on business state" enforceable — not
  as its own rule, but as a side effect of removing the ability to import
  the thing you'd branch on.
- **Not mechanically checkable:** whether a Rule's *name* actually matches
  the predicate it encodes, or whether an Action's orchestration is the
  right shape, is a review judgement — these rules police the namespace
  boundary, not the quality of what's inside it.

## Test preconditions

This is a pure refactor — same behaviour, new home — so Feathers' rule
applies: coverage before refactor. Before relocating a given piece of logic:

- A characterization test must exercise it through the **public** interface
  it currently sits behind (HTTP response, job side effect), never the
  controller method or model directly, so it keeps passing once the logic
  moves. Write it first (`testRetrofit`) if it doesn't exist.
- Where the same rule is **duplicated** across call sites — the exact pain
  this ADR targets — each call site needs its own characterization test
  before consolidation, so a disagreement between copies becomes a
  deliberate merge decision instead of a silent pick.
- Logic inside a model observer, accessor, or scope needs an explicit test
  triggering the lifecycle event before extraction — it may currently only
  be covered incidentally by some unrelated feature's test.

## Steady-state test strategy

Migration coverage above is about not breaking anything on the way in. This
is what the harness looks like for good, once the service layer exists as
its own safe space — and it changes the pyramid, not just adds to it.

- **A Rule is a unit test.** No framework boot, no database, table-driven
  over inputs. This is the cheapest test category we have, and a Rule's
  whole reason for existing is to make that category reachable for logic
  that previously couldn't get there.
- **An Action is a component test.** Real Rules, its persistence port faked
  or real depending on cost, no HTTP. It proves orchestration — which Rules
  fire, in what order, with what side effect — not the rules' own edge
  cases, which already live one level down.
- **The acceptance test for the endpoint narrows to a wiring proof:** DTO in,
  Action called, response mapped correctly. One test, not one per business
  rule variant.

The pyramid effect: today, a business-rule edge case is often only reachable
through an acceptance test, because the HTTP round trip was the only seam
that could reach it — not because the assertion needed that altitude. Once
the Rule is its own class, that coverage belongs at unit level. **Pruning
the old acceptance-level variant tests is a required step of each
migration, not a side effect assumed to happen later** — landing the unit
test without deleting the acceptance test it superseded just makes the
suite slower for no added confidence. Name the deletion in the PR, the same
way the test-design skill allows naming a throwaway driver test's removal.
