

# What is a port and what is an adapter

 The port is literally just a signature — no body, no logic, "dumb." It's a promise, not code that runs.
- You call $this->consignmentsPort->doSomething() in your class.
- Laravel's container looks up the binding ("when someone asks for ConsignmentsPort, give them a HttpConsignmentsAdapter") and hands you the adapter object instead.
- So at runtime ->doSomething() actually executes HttpConsignmentsAdapter::doSomething() — that's where the real HTTP call + mapping logic lives.

You're calling the interface by name, but the object underneath is always the adapter. The port is just the agreed shape both sides (your class, and whatever implements it) have to honor.



# What is the providers folder doing?

The Providers\ folder holds the class that does the binding — it's the wiring/glue that tells Laravel's container "when something asks for ConsignmentsPort, give it HttpConsignmentsAdapter."

From the PR, BoxDeliveryServiceProvider (in App\BoxDelivery\Providers\) contains something like:

public function register() {
    $this->app->bind(ConsignmentsPort::class, HttpConsignmentsAdapter::class);
}

It has to live inside the domain's own folder (not a shared/global provider) because binding it from outside would require naming that domain's internal Ports\/Adapters\ classes from elsewhere — which the architecture rules forbid (cross-domain boundary violation). So each domain owns and registers its own wiring.



# What is the contracts folder?

Contracts\ is what a domain exposes to other domains — the only legal doorway in.

Where a port is domain-internal ("what I need from the outside"), a contract is domain-external ("what I offer to others"). If domain B wants something from domain A, B may only depend on A\Contracts\... — never reach into A\Ports\, A\Adapters\, or A\Application\ directly. That's what CrossDomainBoundaryTest enforces.

Sub-folders seen in the PR:
- Contracts\Data\ — DTOs the domain returns/accepts across the boundary (currently empty everywhere — flagged as the real gap, since without these, methods just pass around raw array).
- Contracts\Exceptions\ — exceptions the domain may throw that callers are allowed to catch.

So: Port = domain talking to the outside world (infra). Contract = other domains talking to this domain (inter-domain API).

 ## so here are we saying if someone wants to talk to the boxdelivery domain they would get the DTO from the contract data folder and import that and use that to get the data?

Almost — one nuance: the DTO in Contracts\Data\ is just the shape of data, not the entry point itself. You still need a method (on some class the domain exposes) that actually returns/accepts that DTO.

So the flow would be: another domain imports and calls a domain-boundary method (e.g. something like BoxDelivery\Contracts\ConsignmentsApi::getConsignments($orderId)), and that method's signature says it returns a Contracts\Data\ConsignmentDto — that's what the caller imports to know the shape they're getting back. They're not "reaching into the folder to get data" — they call the exposed method, and the DTO is what travels back across that call.

Right now in this PR, Contracts\Data\ is empty for every domain (just a .gitkeep) — no domain has actually built that exposed entry point + DTO yet. That's flagged in the PR as the next real piece of work.



 
# what is opaque resolution rule

It catches ways of getting an object that bypass normal dependency injection, so the dependency doesn't show up as a typed reference anywhere — meaning none of the other rules can see it or check it.

Normally, if MyClass needs SomeEloquentModel, you'd see it in a type-hint (constructor param, method signature) — that's a visible "edge" in the dependency graph, so the Layer/Domain rules can inspect and flag it if it's not allowed.

But you can get the same object without ever naming its type, e.g.:

`$model = app('SomeEloquentModel');       // string-keyed container lookup
`$value = Route::input('order_id');       // pulls straight from the request`

No class name appears anywhere PHPStan's static analysis can see, so no rule fires — the coupling exists at runtime but is invisible on paper.

Why it matters (per the PR): without this rule, someone "fixing" a violation could just replace a named model reference with a string container lookup — same bad coupling, but now invisible, so the violation count in the baseline goes down while the actual architecture gets worse. The commit notes roughly half of all container resolution in app/ is already done this string-keyed way.

NoOpaqueDependencyResolutionRule is an AST-level check specifically hunting for these string-keyed/opaque resolution patterns and flagging them as violations too, so that route is closed off.


# what are other layers

Driving (inbound) side — where requests enter:
- InboundAdapter — not just Http\Controllers, but also Http\Middleware, Console (CLI commands), Listeners, Observers, Jobs. Anything that's an entry point into the app counts as this layer. There's deliberately no "driving port" — an inbound adapter calls a use case directly, no interface in between on this side.

Inward layers (the new greenfield ones):
- Application — the use-cases (e.g. GetConsignmentsForOrder). May depend on: its own layer, Ports, Contracts (of any domain).
- Ports — driven ports, internal to the domain, may only depend on other Ports.
- Contracts — the published surface for other domains, may only depend on other Contracts.

Driven (outbound) side:
- Adapters — implements the ports, may depend on Ports, Contracts, Adapters, and LegacyPersistence, plus transport libs (Guzzle etc.).
- LegacyPersistence — Eloquent models + legacy per-model repositories. Tightly capped: may only depend on itself and persistence mechanics — nothing else. This is deliberately used as a discovery tool: anything a model currently reaches (a Service, Presenter, Validator, facade) shows up as a violation, surfacing business logic that's leaked into the persistence layer.

Cross-cutting rule: persistence_access — only Adapters and LegacyPersistence may touch the DB/Eloquent at all. Every other layer (including InboundAdapter, Services\, Packages\) touching persistence directly is a violation — that's the entire pre-existing baseline.

Explicitly exempt from everything: Database\Factories, Tests, Core\Tests, CoreTests — not treated as debt, since a factory's job is to build models and tests legitimately reach across domains.

So the layer axis has 6 named layers total: InboundAdapter, Application, Ports, Contracts, Adapters, LegacyPersistence — plus the pre-existing Services\/Packages\/BoundedContexts\ code, which is deliberately left unclassified (not mapped to any layer) rather than shoehorned in, since none of it coheres with the target shape yet.



ask about http layer and ask about h-ow controller should call a port 

