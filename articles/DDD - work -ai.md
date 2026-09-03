---
tags:
  - result-domain-driven-design
  - hexagonal-architecture
  - aggregates
  - repository-pattern
  - port-adapters/result
---

# DDD — Bringing Domain-Driven Design to a Legacy Service

Context: these are working notes on introducing DDD patterns into an existing legacy service, not a greenfield build. The core tension throughout is: how do we introduce clean boundaries (domain layer, ports/adapters) without a big-bang rewrite, and how do we stop persistence/delivery concerns leaking into domain logic.

## 1. Layering: Controller → Service → Domain

- **Controller should call the Service layer, not the domain or repository directly.** Controllers are inbound adapters — they translate an HTTP/queue/event trigger into a call against the service layer's API. They should contain no business logic.
- **Service layer** sits between controllers and the domain model. It orchestrates use cases: load an aggregate (via a repository port), invoke behaviour on it, persist the result, raise any events. It should stay thin — the "what happens in what order," not "how do we decide."
- Rule of thumb for the legacy migration: any logic currently sitting in a controller or DB-facing class that makes a *decision* (not just data shuffling) is a candidate to move into the domain layer.

Open question to resolve with an example: does *every* controller action go through a service, or can trivial reads bypass it? Worth deciding as a team convention rather than case-by-case.

## 2. Ports and Adapters (Hexagonal Architecture)

- A **port** is an interface owned by the domain/application layer, describing what it needs (or offers) without saying how.
  - **Inbound port**: how the outside world drives the application (e.g. "PlaceOrder" use case interface). Controllers implement/call these.
  - **Outbound port**: what the application needs from the outside world (e.g. "OrderRepository", "NotificationSender"). The domain/service layer depends on these interfaces only.
- An **adapter** implements a port for a specific technology — it "turns a port into something concrete": a Postgres repository adapter, an SNS/event-bus publisher adapter, an HTTP client adapter for a delivery service.
- Practical benefit for a legacy service: we can introduce a port with a *thin adapter wrapping the existing legacy persistence code* first, without changing that code. This decouples the domain from persistence immediately, and lets us swap/refactor the adapter later without touching domain logic.
- **This should be the main focus of the migration work**: get ports and adapters in place around the domain boundary before trying to perfect the domain model itself. Sequencing matters more than purity.

## 3. Aggregates

- An aggregate is a cluster of entities/value objects treated as a single consistency boundary — one aggregate root is the only object referenced from outside, and it's the only thing that can be loaded/saved atomically.
- **When to use one**: when a group of objects has invariants that must always hold together (e.g. an order and its line items must stay consistent — you shouldn't be able to save one without the other in a broken state). If two objects can change independently with no shared invariant, they're probably separate aggregates, not one.
- Once we have an aggregate identified, **the repository should load/save the whole aggregate, not individual parts of it** — this is what "when we have an aggregate we'd use that instead" was pointing at: instead of ad-hoc queries/updates against individual tables, all reads/writes for that consistency boundary go through the aggregate's repository.
- Legacy migration heuristic: look at existing transactions/locking in the current code — wherever multiple tables are updated together "to stay consistent," that's a strong signal for an aggregate boundary.

## 4. Repository Pattern

- The repository is an **outbound port** that abstracts persistence for an aggregate: `save(aggregate)`, `getById(id)`, etc. The domain/service layer only ever talks to this interface.
- **The persistence layer is hard to design well before the domain layer exists** — schema and query shape should follow the domain model's needs, not the other way round. In a legacy context this is usually backwards (schema came first), so expect friction: the repository adapter may need real translation logic between the domain model and the existing schema, at least until the schema itself can be evolved.
- Access persistence **via the port/adapter, never via a concrete implementation** reached directly from service or domain code. If a service class is `new`-ing up a concrete DB client or ORM entity itself, that's the smell to fix first.

## 5. Outbox Pattern

- Used where a domain operation needs to *both* persist a state change *and* reliably publish an event (e.g. "OrderConfirmed") — without two-phase commit across a DB and a message broker.
- Mechanism: write the event to an `outbox` table in the **same transaction** as the aggregate's state change, then a separate relay process/poller publishes outbox rows to the event bus and marks them sent.
- Relevant here because "cross domain communication" and "events — event bus notification" both eventually need this guarantee: if a legacy service publishes events as a side effect of an update, and the publish fails independently of the DB commit, we get inconsistency. Outbox is the standard fix.
- Worth flagging as a dependency: introducing the outbox pattern implies we already have a transactional boundary around the aggregate (see §3) — so aggregate boundaries should be sorted before implementing outbox.

## 6. Cross-Domain / Outbound Communication

Three distinct outbound concerns, worth keeping conceptually separate even if they end up looking similar in code:

1. **Repository → DB**: internal persistence, private to the owning domain.
2. **Cross-domain, in-process code calls**: calling into another bounded context's code directly (in-process). Should still go through a port — treat the other domain as if it were external, to avoid coupling to its internals.
3. **Outside/external services** (e.g. a delivery service): calls out to systems owned by other teams/bounded contexts entirely. Definitely an adapter behind a port, likely with its own resilience concerns (timeouts, retries) that don't belong in the domain.
4. **Events / event bus notifications**: async, one-to-many, decoupled — for facts that happened, not commands. Pairs with the outbox pattern (§5) for reliability.

The common thread: the domain/service layer should not be able to tell the difference between these at the interface level — they're all just outbound ports. The *adapter* is where the different technology/reliability concerns live.

## 7. Worked Example — for the ADR

Plan is to ground all of the above in one concrete worked example, candidates:
- **Consignment controller**, or
- **Box delivery**

Whichever is picked, the ADR should walk through:
- What's the aggregate (and why — what invariant does it protect)?
- What's the repository port + legacy-backed adapter look like?
- Where does cross-domain communication happen, and via what port?
- Does this use case need an outbox (does it emit an event alongside a state change)?
- What does the controller → service → domain call chain look like end to end?

This example becomes the reference pattern other developers copy when migrating the next slice of the legacy service, so it's worth over-investing in getting this one right rather than trying to migrate everything at once.

## Open Questions / Follow-ups

- Do all controller actions route through the service layer, or can trivial reads skip it?
- What's the criteria for "this is one aggregate" vs "these are two aggregates that reference each other by id"?
- Pick consignment vs box delivery for the ADR example and write it up.
- Confirm whether outbox is needed everywhere events are published, or only where the event must not be lost (some notifications may be best-effort).
