---
tags:
  - result-machine-learning
  - test-layers
  - unit-testing
  - functional-testing
  - e2e-testing/result
---

# Gousto2-Core · Test Layers

How testing is structured across the repo — from pure-unit up to full end-to-end, plus the specialised guards alongside.

## At a glance

| Metric | Value |
|---|---|
| Behavioural layers | 5 |
| Test runners | 3 |
| Domains | 8 |
| Layout conventions | 2 |

## The Pyramid — fast & broad at the base, slow & few at the top

| Tier | Name | Description |
|---|---|---|
| 1 (base) | **Unit-fast** | pure PHP · no framework · no DB |
| 2 | **Unit-slow** | boots Laravel · real DB · Mockery |
| 3 | **Bounded-context** | a domain module as a unit |
| 4 | **API** | in-process HTTP · real DB |
| 5 (top) | **Functional** | out-of-process E2E · closed-box |

## Behavioural layers

### 1 · Unit-fast
**Tooling:** Codeception · FastUnitTester

- **Layer:** Pure PHP. Laravel *disabled*, no database.
- **Purpose:** Isolated logic — value objects, domain services, calculations. Fast because they skip the slow Laravel boot.
- **Run:** `SUITE=box-ordering-unit-fast`
- **Characteristics:** ~~Laravel~~, ~~DB~~, **Asserts** (on)

### 2 · Unit-slow
**Tooling:** Codeception / PHPUnit

- **Layer:** Boots Laravel, real DB access, factories, Mockery for collaborators.
- **Purpose:** Components needing the framework/container or DB — models, repositories, controllers in isolation. The integration-ish middle.
- **Run:** `SUITE=box-ordering-unit-slow`
- **Characteristics:** **Laravel**, **Database**, **Factory**, **Mockery** (all on)

### 3 · Bounded-context
**Tooling:** PHPUnit · bounded-contexts

- **Layer:** Across a bounded context's internal pieces.
- **Purpose:** Verify a domain module works as a unit through its own boundaries.
- **Run:** `RUNNER=phpunit SUITE=bounded-contexts`
- **Path convention:** `tests/<Domain>/Phpunit/BoundedContexts`

### 4 · API
**Tooling:** Codeception · ApiTester

- **Layer:** Boots Laravel + REST module; drives real HTTP endpoints against a real DB, in-process.
- **Purpose:** Endpoint request→response contract — routing, auth, serialization, persistence. External services still mocked.
- **Run:** `SUITE=menu-planning-api`
- **Characteristics:** **Laravel**, **REST**, **Database**, **Auth** (all on)

### 5 · Functional
**Tooling:** functional-tests/ · separate project

- **Layer:** Highest — full system, closed-box. Hits the running service from outside over HTTP; state set up via `/_test/` tooling drivers.
- **Purpose:** End-to-end behaviour of real user flows (e.g. order placement). Own `*_functional` schemas, isolated from unit DBs.
- **Run:** `FUNCTIONAL_TESTS="OrderPlacementTest"`
- **Tags:** closed-box, `/_test/` drivers, `gousto_functional`

## Specialised guards (not the pyramid)

### Characterisation
Pins *current* behaviour of legacy code — not asserting it's correct, just capturing what it does so you can refactor safely.

- **Tag:** `SUITE=characterisation`

### Acceptance
Top-level scenario suite over the booted app.

- **Tag:** `SUITE=acceptance`

### Architecture
Not behavioural. PHPStan + PHPat enforce structural rules — e.g. `App\<Domain>` internals reachable only through a declared surface (ADR 0004).

- **Tag:** `src/architecture/`

### Python components
Separate from the PHP stack — for the Python Step Function components.

- **Tags:** `make dev-test-orderfinaliser`, `make dev-test-cutofforchestrator`

### Two layouts (live in parallel)
- **Domain-first** (new): `tests/<Domain>/{Phpunit,Codeception}/…`
- **Type-first** (legacy): `tests/{phpunit,codeception}/…`

### DB isolation
One MySQL on :3306. Unit uses `gousto`/`orders`, functional uses `*_functional`. Only one of each type runs at a time.

## Cheat sheet

```bash
# Everything
make dev-test

# By runner
make dev-test RUNNER=phpunit
make dev-test RUNNER=codeception
make dev-test RUNNER=functional

# A whole suite (discover names: make dev-list-test-suites)
make dev-test RUNNER=codeception SUITE=box-ordering-unit-slow
make dev-test RUNNER=phpunit     SUITE=bounded-contexts

# A specific path (relative to src/, type auto-detected)
make dev-test TESTS="tests/BoxOrdering/Codeception/UnitSlow/Controllers"

# A functional test — pass the METHOD or CLASS name, not a path
make dev-test FUNCTIONAL_TESTS="testReplacingRecipesOnOrder"

# Reseed test DB after data-consistency errors
make dev-init-test-db
```

---

*Generated for Gousto2-Core · sources: `GUIDELINES.md`, `run-tests` skill, `src/tests/codeception/*.suite.yml`, `src/architecture/`*
