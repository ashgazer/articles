---
tags:
  - result-machine-learning
  - software-engineering
  - design-principles
  - technical-debt
  - system-architectures/result
---

# Software Laws, Principles & More Terms — Companion Guide

A companion to the concepts guide. Two parts: the **named "laws" and principles** people love to cite, then a second batch of **terms worth adding**. As before, the focus is on what each *means in conversation* and why it gets invoked.

---

## Part 1: The "laws" people cite

Engineers throw these around partly as shorthand and partly for fun. You don't need to memorise them — but recognising the name and the gist means you're never lost when someone says "well, that's just Conway's Law."

### The genuinely useful ones (you'll hear these)

**Murphy's Law** — *Anything that can go wrong, will go wrong.* Invoked when planning for failure: "assume Murphy's Law and add a retry."

**Conway's Law** — *A system's design ends up mirroring the communication structure of the org that built it.* Four teams building a compiler produce a four-pass compiler. Comes up in architecture and re-org discussions: "our tangled service boundaries are a Conway's Law problem."

**Hyrum's Law** — *With enough users, every observable behaviour of your API will be depended on by someone*, no matter what your contract says. The reason "harmless" changes break someone. Cited when debating whether a change is *really* backwards-compatible.

**Goodhart's Law** — *When a measure becomes a target, it stops being a good measure.* Optimise for "lines of code" or "tickets closed" and people game it. Comes up around metrics and OKRs.

**Postel's Law (Robustness Principle)** — *Be conservative in what you send, liberal in what you accept.* Guidance for designing tolerant APIs and parsers. (Note: it's fallen somewhat out of fashion — being *too* liberal in what you accept causes its own problems, so you'll also hear it criticised.)

**Brooks's Law** — *Adding people to a late software project makes it later.* Because onboarding and communication overhead outweigh the extra hands. Cited when someone suggests "just throw more devs at it."

**Hofstadter's Law** — *It always takes longer than you expect, even when you account for Hofstadter's Law.* The self-referential joke about estimation. Pulled out when a task overruns (again).

**Parkinson's Law** — *Work expands to fill the time available for its completion.* Argument for tight deadlines and small scopes.

**The Ninety-Ninety Rule** — *The first 90% of the code takes 90% of the time; the remaining 10% takes the other 90%.* A joke about how the "nearly done" part drags forever. Cited when a "quick finish" isn't.

### The design/quality ones

**Cunningham's Law** — *The best way to get the right answer online isn't to ask a question — it's to post the wrong answer.* People will rush to correct you. Named after Ward Cunningham (who also gave us "technical debt").

**Kernighan's Law** — *Debugging is twice as hard as writing the code. So if you write the cleverest code you can, you are — by definition — not smart enough to debug it.* The canonical argument against being too clever.

**Wirth's Law** — *Software gets slower faster than hardware gets faster.* Why your new laptop still feels sluggish.

**Zawinski's Law** — *Every program attempts to expand until it can read email.* A jab at feature creep — apps keep bloating until they do everything.

**Gall's Law** — *A complex system that works invariably evolved from a simple system that worked.* You can't design a working complex system from scratch; you grow it. Argument against big-bang rewrites.

**Pareto Principle (the 80/20 rule)** — *Roughly 80% of effects come from 20% of causes.* 80% of the value from 20% of the features; 80% of bugs from 20% of the code. Used to prioritise.

**Moore's Law** — *Transistor density (roughly, compute power) doubles about every two years.* The historical hardware trend — mostly mentioned now to note that it's slowing down.

### One that connects to "load-bearing" from before

**Chesterton's Fence** — *Don't remove a fence until you understand why it was put there.* If you don't know what a piece of code does, don't delete it assuming it's useless — find out first. This is the principled version of the "that's load-bearing" warning from the last guide: both say *respect the thing you don't fully understand yet.*

---

## Part 2: The principles / acronyms

These are prescriptive — actual design advice, cited to justify or challenge a decision.

**DRY** — *Don't Repeat Yourself.* Avoid duplicating logic; one source of truth for each piece of knowledge. The opposite is jokingly **WET** ("Write Everything Twice"). Note: DRY can be over-applied — sometimes a little duplication beats a bad abstraction.

**KISS** — *Keep It Simple, Stupid.* Prefer the simple solution.

**YAGNI** — *You Aren't Gonna Need It.* Don't build for hypothetical future requirements; build what you need now. Cited to shoot down speculative over-engineering.

**SOLID** — five object-oriented design principles bundled under one acronym. You'll mostly hear individual letters referenced:
- **S** — *Single Responsibility*: a class should have one reason to change.
- **O** — *Open/Closed*: open for extension, closed for modification.
- **L** — *Liskov Substitution*: subtypes must be usable wherever their parent is.
- **I** — *Interface Segregation*: many small interfaces beat one fat one.
- **D** — *Dependency Inversion*: depend on abstractions, not concretes. (This is the theory behind Laravel's "depend on the contract" advice.)

**Separation of concerns** — keep distinct responsibilities in distinct places (HTTP handling separate from business logic separate from persistence).

**Law of Demeter (Principle of Least Knowledge)** — *only talk to your immediate collaborators.* Don't reach through objects: `$order->customer->address->postcode` is a Demeter violation ("train-wreck" chaining).

**Principle of Least Astonishment** — code/APIs should behave the way people reasonably expect. A method called `getUser()` shouldn't also delete something.

**Composition over inheritance** — prefer building behaviour by combining small parts over deep inheritance hierarchies. (In PHP this often means traits and injected services rather than extending base classes.)

**Rule of Three** — don't extract an abstraction until you've seen the pattern *three* times. Two occurrences might be a coincidence; three is a pattern.

**The Boy Scout Rule** — *leave the code a little cleaner than you found it.* Small, opportunistic improvements as you pass through.

**"Premature optimization is the root of all evil"** (Knuth) — don't optimise before you've measured and know it matters. Frequently cited (and frequently misused to avoid *any* thought about performance).

**"Make it work, make it right, make it fast"** — the order to do things in: correctness first, clean design second, speed last.

---

## Part 3: More terms worth having

These round out the backend/Laravel vocabulary from the first guide.

### The famous joke-quote you'll definitely hear

> "There are only two hard things in computer science: cache invalidation and naming things." — Phil Karlton

People quote it constantly (often adding "…and off-by-one errors" as a bonus joke). Worth recognising.

### Operational / deployment terms

| Term | Meaning |
|---|---|
| **Feature flag / toggle** | A switch to turn functionality on/off without redeploying — for gradual rollout or quick disabling. |
| **Canary release** | Ship a change to a small % of traffic first, watch for problems, then roll out fully. |
| **Blue-green deployment** | Two identical environments; switch traffic from old ("blue") to new ("green") for zero-downtime releases and instant rollback. |
| **Rollback / roll forward** | Undo a bad deploy (back) vs. push a fix on top (forward). |
| **Blast radius** | How much breaks if this thing fails — a "big blast radius" change is high-risk (close cousin of "load-bearing"). |
| **Graceful degradation** | When part of the system fails, the rest still works in a reduced way rather than everything collapsing. |
| **Circuit breaker** | A pattern that stops hammering a failing dependency, giving it time to recover. |

### Data / concurrency terms

| Term | Meaning |
|---|---|
| **Race condition** | A bug where the outcome depends on the unpredictable timing of concurrent operations. |
| **Deadlock** | Two processes each waiting on the other, so neither proceeds — everything stalls. |
| **Eventual consistency** | Data isn't in sync everywhere *immediately*, but converges shortly. Common in distributed systems. |
| **ACID** | The guarantees of a reliable database transaction: Atomicity, Consistency, Isolation, Durability. |
| **Idempotency key** | A token that ensures retrying a request (e.g. a payment) doesn't run it twice. |
| **Cache stampede / thundering herd** | Many requests all miss the cache at once and slam the database simultaneously. |

### Change-management terms

| Term | Meaning |
|---|---|
| **Breaking change** | A change that forces consumers of your code/API to update. |
| **Backwards compatibility** | New version still works with old callers/data. |
| **SemVer (Semantic Versioning)** | The `MAJOR.MINOR.PATCH` version scheme — bump MAJOR for breaking changes, MINOR for features, PATCH for fixes. |
| **Deprecation** | Marking something as "don't use this, it's going away" before actually removing it. |

### Everyday dev-culture slang

| Term | Meaning |
|---|---|
| **Yak shaving** | Doing a frustrating chain of unrelated sub-tasks just to get to the thing you actually wanted to do. |
| **Rubber-duck debugging** | Explaining your problem out loud (to a rubber duck, or a colleague) and solving it mid-explanation. |
| **Heisenbug** | A bug that disappears or changes when you try to observe it. |
| **Bus factor** (or truck factor) | How many people could get "hit by a bus" before the project is in trouble — i.e. how concentrated the knowledge is. A bus factor of 1 is dangerous. |
| **Greenfield vs brownfield** | Building fresh from nothing (greenfield) vs. working within an existing, constrained system (brownfield). |
| **Spaghetti code / big ball of mud** | Tangled, structureless code with no clear boundaries. |
| **Paved road / golden path** | The blessed, well-supported way to do something in your org — stray off it and you're on your own. |
| **Dogfooding** | Using your own product internally ("eating your own dog food"). |
| **Toil** | Repetitive manual operational work that ought to be automated. |
| **Flaky test** | A test that sometimes passes and sometimes fails without the code changing — usually a timing/ordering issue. |

---

## How to use this

Same as before — don't cram. The named laws especially are the kind of thing that clicks the moment you hear one in context and glance back here. If you want, next time your team drops one you don't recognise, note it down and I can slot it into the guide with the same treatment.
