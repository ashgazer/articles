---
tags:
  - machine-learning
  - service-container
  - inversion-of-control
  - autoloading
  - psr-4
---

# Laravel & Framework Concepts — grounded in Gousto2-Core

## From "what is a facade" to the whole picture

A framework like Laravel takes control of *running* your app and asks your code to fill in the blanks. These are the concepts that make that work. Read them top to bottom — each builds on the one before. Every code example is real, taken from this repo.

---

## Foundations

### 1. Framework vs Library (Inversion of Control)
*the mental shift*

**One-liner:** A library is code YOU call. A framework is code that calls YOU.

**Plain English**

With a plain **library**, your code is in charge: you decide when to call `parseDate()` or `sendEmail()`.

With a **framework** like Laravel, the framework is in charge. It boots, listens for a web request, and then calls *your* code (a controller, a model) at the right moment. You don't write the `main()` loop — Laravel owns it and hands control to your pieces. This flip is called **Inversion of Control**.

This is why so much of Laravel is about *registering* things ("here's my route", "here's how to build my service") rather than calling them. You hand Laravel the parts; it decides when to run them.

**Analogy**

**Restaurant kitchen.** A library is your own kitchen — you cook when you want. A framework is working as a line cook in someone else's restaurant: the kitchen runs the show, tickets come in, and the head chef calls "fire the pricing station!" — you just need your station ready to respond.

**Real code**

```php
// You almost never write this loop. Laravel does, in public/index.php:
$app = require 'bootstrap/app.php';
$response = $app->handle($request);  // Laravel runs everything…
$response->send();                  // …and calls YOUR controller somewhere inside
```

In this repo: Your whole app lives under `src/` and runs through Laravel's bootstrap — you never see the loop, you just add routes, controllers and providers.

**How it connects →** Everything below is a "blank" the framework asks you to fill in: **where** requests go (routing), **how** to build services (providers/container), and **what** shortcuts to expose (facades).

---

### 2. Namespaces & Autoloading (PSR-4)
*how files become classes*

**One-liner:** A namespace is a class's full address; PSR-4 maps that address to a folder so PHP finds the file automatically.

**Plain English**

PHP needs to know which file defines `App\Pricing\...\VATFacade`. Old PHP made you `require` files by hand. **Autoloading** removes that: you just *use* a class and PHP loads the file on demand.

**PSR-4** is the rule that makes it predictable: a namespace *prefix* maps to a *folder*, and the rest of the namespace mirrors the subfolders. So the class name literally tells you where the file is.

This is exactly why the Pricing-relocation PR was so mechanical: moving files meant rewriting every `namespace` line and every `use` line to keep the address-to-folder mapping intact.

**Analogy**

**Postal address.** The namespace `App\Pricing\Http\Controllers\PricesController` is like "UK / London / Pricing St / no. 12". PSR-4 is the rule that "App\ means the `app/` folder", so the postman (PHP) walks the rest of the path to the exact file.

**Real code**

```php
// src/composer.json — the PSR-4 map (prefix → folder)
"App\\":   "app/",
"Tests\\": "tests/"

// So this class…
namespace App\Pricing\Http\Controllers;  // App\ → app/
class PricesController { }
// …must live at: src/app/Pricing/Http/Controllers/PricesController.php
```

In this repo: The relocation removed old prefixes (`BoundedContexts\`, `Gousto\VAT\`) from `src/composer.json` because every Pricing class now lives under the single `App\` → `app/` mapping.

**How it connects →** Once PHP can *find* your classes, the next question is how they get *built* and wired together — that's the container.

---

## How the app is wired

### 3. The Service Container
*the heart of Laravel*

**One-liner:** A central object that knows how to build and hand out your classes, fully wired, on request.

**Plain English**

Almost every Laravel concept sits on top of one object: the **service container** (you'll see it as `$this->app` or `app()`).

You teach it: *"when something needs X, build it like this."* Later you ask it for X and it returns a ready-to-use instance — assembling any sub-parts X depends on. Two verbs matter: **bind** (teach it) and **resolve** / **make** (ask it).

Why bother? Because it removes the `new` keyword from your business code. Your classes say what they *need*; the container decides how to *build* it. Swap the implementation in one place (e.g. for tests) and everything downstream gets the new one.

**Analogy**

**A coffee machine with presets.** You don't grind beans and steam milk yourself. You press "VAT" and a fully-made VAT service comes out. Someone configured the presets once (a service provider); everyone else just presses the button.

**Real code**

```php
// Teach the container (bind): "VAT" → a VATCalculator
$this->app->bind('VAT', fn() => new VATCalculator(3));

// Ask the container (resolve):
$vat = app('VAT');   // → a ready VATCalculator(3)
```

In this repo: `VATServiceProvider` binds the key `'VAT'`. `PricingServiceProvider` uses richer "contextual" binding: *when* it builds `PricingService`, it gives it a fully hand-assembled `PricingModelFactory`.

**How it connects →** **Binding** and **resolving** are the two halves. The most common way classes get their dependencies *resolved* automatically is Dependency Injection ↓.

---

### 4. Dependency Injection
*how classes get their collaborators*

**One-liner:** Instead of a class creating what it needs, it asks for those things in its constructor and the container supplies them.

**Plain English**

A "dependency" is just another object your class needs to do its job (a calculator, a repository, a mapper).

**Without DI:** your class does `new VATCalculator(3)` inside itself — now it's welded to that exact class and hard to test.

**With DI:** your class lists what it needs as constructor parameters. Laravel sees the type-hints, resolves each one from the container, and "injects" them when it builds your class. Your class never says `new`.

This is the *why* behind the container: DI is the container doing its job automatically based on type-hints.

**Analogy**

**Ordering ingredients vs. owning a farm.** Without DI, every recipe runs its own farm to grow tomatoes (rigid, can't substitute). With DI, the recipe just says "I need tomatoes" and the kitchen delivers them — swap to organic tomatoes once, every recipe benefits.

**Real code**

```php
// PricesController asks for what it needs — never "new"s them:
public function __construct(
    private PricingService $pricing,
    private PriceBreakdownToPricesArrayMapper $mapper,
) {}
// Laravel reads the type-hints, builds each from the container, injects them.
```

In this repo: Real example: `src/app/Pricing/Http/Controllers/PricesController.php` imports `PricingService`, the mapper and `PricesSpanService` and receives them — the container wires the graph defined in `PricingServiceProvider`.

**How it connects →** For DI to inject the *right* implementation, someone must register the bindings. That registration lives in Service Providers ↓.

---

### 5. Binding & Resolving (incl. contextual binding)
*teaching vs asking*

**One-liner:** Binding = telling the container how to build a key/class. Resolving = asking for it. Contextual binding = "build it differently depending on who's asking."

**Plain English**

Three flavours you'll see in this repo:

1. **Simple bind:** "key → factory". `bind('VAT', fn() => new VATCalculator(3))`.
2. **Auto-resolution:** if a class has no special wiring, the container just `new`s it and recursively injects its constructor dependencies. No bind needed.
3. **Contextual bind:** "*when* class A needs B, give it *this specific* B." Used when one consumer needs a hand-tuned version.

**Analogy**

**Custom orders.** Simple bind = "coffee always means a flat white". Contextual bind = "when *the kids' table* orders coffee, make it a decaf" — same request, different build depending on context.

**Real code**

```php
// Contextual binding from PricingServiceProvider:
$this->app->when(PricingService::class)   // WHEN building PricingService
    ->needs(PricingModelFactory::class)    // and it NEEDS a factory
    ->give(fn() => /* a fully assembled PricingModelFactory */);
```

In this repo: See the three `when()->needs()->give()` blocks in `PricingServiceProvider::register()` — they hand `PricingService` its `BoxPricing`, `PricingModelFactory`, and feature-flag switcher.

**How it connects →** All this binding code has to live *somewhere* that runs at boot. That somewhere is a Service Provider.

---

### 6. Contracts & Interfaces
*depend on abstractions*

**One-liner:** An interface lists the methods a class must have without saying how; you bind the interface to a concrete class so the rest of the app depends on the role, not the implementation.

**Plain English**

An **interface** (Laravel calls them **contracts**) is a promise: "any class implementing me has these methods." Your code type-hints the *interface*, never the concrete class.

A provider then **binds** the interface to a real implementation. Swap that one binding (e.g. a fake in tests) and every consumer gets the new version — nothing else changes. This is DI and the container paying off together.

**Analogy**

**A job description vs. the person hired.** The role ("must answer the phones") is the interface; you can hire, fire or swap any person (implementation) who fits the description, and the company carries on unaffected.

**Real code**

```php
// Code depends on the interface…
public function __construct(private BoxRepositoryInterface $boxes) {}

// …a provider binds it to a concrete class:
$this->app->bind(BoxRepositoryInterface::class, EloquentBoxRepository::class);
```

In this repo: Interfaces live in `src/app/Models/Interfaces/` (e.g. `BoxRepositoryInterface`). `PricingServiceProvider` resolves one with `app(BoxRepositoryInterface::class)` when wiring the marginal-pricing graph.

**How it connects →** Interfaces get bound to implementations *somewhere* — and that somewhere is a Service Provider ↓.

---

### 7. Service Providers
*where wiring lives*

**One-liner:** The boot-time classes where you register bindings and set things up. They're the "assembly manual" for your app.

**Plain English**

A **service provider** is a class with (mainly) two methods:

- `register()` — add bindings to the container. *Only* wiring here, nothing that needs other services yet.
- `boot()` — run setup after *all* providers have registered (e.g. event listeners, routes).

Laravel runs every provider listed in `config/app.php` at startup. That list is literally the app's wiring index — which is why this relocation PR had to edit `config/app.php` when the provider class names moved.

**Analogy**

**The kitchen's opening checklist.** Before service starts, someone configures every station: "espresso machine → here", "VAT preset → VATCalculator". Providers are that opening routine; once done, the kitchen (your app) can take orders.

**Real code**

```php
class VATServiceProvider extends ServiceProvider {
    public function register() {
        $this->app->bind('VAT', fn() => new VATCalculator(3));
    }
}
// Registered in config/app.php 'providers' => [ … VATServiceProvider::class … ]
```

In this repo: This PR repointed `PricingServiceProvider` and `VATServiceProvider` in `src/config/app.php` to their new `App\Pricing\…` namespaces. The *logic* didn't change — only the addresses.

**How it connects →** Providers fill the container's labelled boxes. A **facade** is a shortcut for reaching into one of those boxes ↓.

---

### 8. Facades
*static-looking shortcuts*

**One-liner:** A facade is a thin proxy that lets you call a container service with `ClassName::method()` syntax. It is NOT the implementation — it forwards to it.

**Plain English**

A facade like `VAT` has *no real methods*. When you call `VAT::calculate($x)`:

1. PHP can't find a static `calculate()` on the facade →
2. Laravel's `__callStatic()` kicks in →
3. It resolves the container key from `getFacadeAccessor()` (here, `'VAT'`) →
4. It forwards your call to the real object (the `VATCalculator`).

So a facade is pure ergonomics: a static-looking nickname for "resolve this service and call it". The single string key (`'VAT'`) is the only thing linking the facade to its service.

**Analogy**

**Hotel front desk.** You say "send this to room VAT." The desk (facade) does nothing itself — it looks up who's in room `'VAT'` and passes your message to them. The actual work is done by the guest (the `VATCalculator`).

**Real code**

```php
class VATFacade extends Facade {
    protected static function getFacadeAccessor() {
        return 'VAT';   // ← the container key it forwards to
    }
}
// Now anywhere:  VAT::calculate($amount)  →  app('VAT')->calculate($amount)
```

In this repo: **Are facades always tied to a provider?** They're always tied to a container *binding*. A provider is the usual place that binding is made (here `VATServiceProvider` binds `'VAT'`) — but a facade whose accessor returns a class name resolves with no provider at all. The alias `'VAT' => VATFacade::class` in `config/app.php` is what lets you write `VAT::` without a `use`.

**How it connects →** Facade, provider and container are the "wiring" trio. Next: how an actual HTTP request flows through the wired app ↓.

---

## Handling a request

### 9. The Request Lifecycle
*how a request flows*

**One-liner:** A request enters, passes through middleware, hits a route → controller, which returns a response that flows back out.

**Plain English**

When someone calls `GET /prices`, roughly this happens:

**Request** → **Middleware** (auth, etc.) → **Router** matches the URL → **Controller** method runs (with its dependencies injected) → returns a **Response** → middleware on the way out → sent to the client.

Everything earlier (container, providers, facades) is the *setup* that happens once at boot. The lifecycle is what happens *per request*, reusing all that wiring.

**Analogy**

**Airport for a passenger (the request).** Security checks (middleware) → gate assignment (routing) → the flight itself (controller) → you arrive (response). Same airport infrastructure (the booted app) serves every passenger.

**Real code**

```php
// The journey of GET /prices in this repo:
/prices  →  middleware  →  router (routes/web.php)
        →  App\Pricing\Http\Controllers\PricesController@index
        →  returns JSON price breakdown  →  back to client
```

In this repo: The route is defined in `src/routes/web.php`; this PR updated it to point at the moved controller's new full class name.

**How it connects →** The next four concepts are the stops on that journey: routing, controllers, middleware, then the data layer.

---

### 10. Routing
*URL → code*

**One-liner:** Routes are the table that maps a URL + HTTP method to the code that should run.

**Plain English**

A **route** says "when a `GET` request hits `/prices`, run this controller method." It's a lookup table the router checks on every request.

Routes can also carry a **name** (for generating URLs) and attach **middleware**.

**Analogy**

**A building directory in the lobby.** "Pricing — Floor 3, Room 12." The router reads the URL like a visitor reads the directory and sends the request to the right office (controller).

**Real code**

```php
// src/routes/web.php
Route::get('prices', [
    'as'   => 'get::/prices',                          // route name
    'uses' => '\App\Pricing\Http\Controllers\PricesController@index',
]);
```

In this repo: This exact line changed in the PR — the action string was repointed from the old `PricesController` location to `\App\Pricing\Http\Controllers\PricesController@index`.

**How it connects →** The route names a controller. Let's look at what a controller is ↓.

---

### 11. Controllers
*the request handler*

**One-liner:** A class whose methods handle requests: read the input, call services to do the work, return a response.

**Plain English**

A **controller** is the entry point for your business logic per request. A good controller is *thin*: it shouldn't contain pricing maths itself — it should delegate to services (like `PricingService`) and just shape the response.

Its dependencies arrive via DI (constructor injection), so the controller never builds them.

**Analogy**

**A receptionist.** Takes your request, doesn't do the specialist work themselves — routes it to the right department (services), then hands you back the result neatly packaged.

**Real code**

```php
class PricesController {
    public function index(Request $request) {
        $breakdown = $this->pricing->getPriceBreakdown(/* … */);
        return $this->mapper->map($breakdown);  // → JSON
    }
}
```

In this repo: `src/app/Pricing/Http/Controllers/PricesController.php` — note it lives under the *Pricing* domain folder now, with its mapper and span-service alongside it.

**How it connects →** Before a request even reaches the controller, it can pass through middleware ↓.

---

### 12. Middleware
*request filters*

**One-liner:** Layers that wrap every request: they can inspect, modify, block, or let it pass — before and after the controller.

**Plain English**

**Middleware** are like a series of checkpoints around your controller. Common jobs: authentication, authorisation, logging, rate-limiting, adding headers.

Each middleware decides whether to pass the request to the next layer (and eventually the controller) or to short-circuit (e.g. "401 Unauthorized"). They run on the way *in* and can also act on the way *out*.

**Analogy**

**Airport security layers.** Boarding pass check, bag scan, passport control — each can wave you through or stop you. Only if you clear them all do you reach the plane (controller).

**Real code**

```php
// Example from this repo — a route guarded by middleware:
Route::put('/internal/menu/v3/{id}', [MenuV3Controller::class, 'update'])
    ->middleware('api.authenticate-and-authorise-action:edit_menus');
```

In this repo: Seen in `src/routes/web.php` just below the `/prices` route — the menu route requires an auth+authorise middleware before its controller runs.

**How it connects →** Once a controller runs, it usually needs data. That's the ORM ↓.

---

### 13. Validation & Form Requests
*reject bad input early*

**One-liner:** Rules that check incoming request data before your controller logic runs; invalid input is rejected automatically with a 422 response.

**Plain English**

Never trust raw input. **Validation** defines rules ("delivery_slot_id is required", "quantity must be an integer") and Laravel enforces them *before* your controller body runs.

For anything non-trivial you move the rules into a **Form Request** class, which also handles authorisation — keeping the controller clean.

**Analogy**

**A bouncer with a guest list.** Requests that don't meet the rules never get past the door (the controller) — they're turned away at the entrance with a clear reason.

**Real code**

```php
// Inline validation inside a controller:
$data = $request->validate([
    'delivery_slot_id' => 'required|string',
    'items'            => 'array',
]);  // fails → automatic 422, controller body skipped
```

In this repo: `src/app/BoxOrdering/Http/Controllers/DayController.php` uses a Form Request; the `/prices` endpoint's required params (e.g. `delivery_slot_id`) are exactly this kind of input contract.

**How it connects →** Controllers are the HTTP front door. There's a second front door that skips HTTP entirely — console commands ↓.

---

### 14. Artisan / Console Commands
*the terminal front door*

**One-liner:** Classes you run from the command line (`php artisan …`) — the same app, entered through the terminal instead of a web request.

**Plain English**

Not everything is triggered by a URL. A **console command** is a class with a `handle()` method you run from the terminal — for backfills, admin tasks, scheduled jobs, one-offs.

It gets the full app: DI, services, models — just no HTTP request/response. Think of it as a controller for the command line.

**Analogy**

**The staff entrance.** Customers arrive through the front door (HTTP); staff use the side door (CLI) to do back-office work. Same building, same equipment inside, different way in.

**Real code**

```php
class GenerateMissingRafCodes extends Command {
    protected $signature = 'raf:generate-missing-codes';
    public function handle() { /* do the work */ }
}
// run with:  php artisan raf:generate-missing-codes
```

In this repo: See `src/app/Console/Commands/` — e.g. `GenerateMissingRafCodes`, `DisableFeatureCommand`, `MovePromotionCodesFromCampaignAToCampaignB`.

**How it connects →** Whether work starts from HTTP or CLI, the things that happen can be broadcast as events ↓.

---

## Events & background work

### 15. Events & Listeners
*announce, then react*

**One-liner:** An event is a message saying something happened (e.g. `SubscriptionCreated`). Listeners are separate classes that react to it. The code firing the event has no idea who is listening.

**Plain English**

This is how Laravel **decouples** "a thing happened" from "everything that should follow."

You **fire** an event (`SubscriptionCreated`). One or many **listeners** each run independently — send an email, create orders, generate a referral code. To add a new reaction you write a new listener and register it; you never touch the code that fired the event.

The wiring lives in an `EventServiceProvider`'s `$listen` map: event ⟶ list of listeners.

**Analogy**

**A town crier.** They shout "royal baby born!" (the event) and walk on. The bakers, bell-ringers and florists each spring into action (listeners) — the crier neither knows nor cares who reacts.

**Real code**

```php
// EventServiceProvider: one event → many listeners
protected $listen = [
    SubscriptionCreated::class => [
        FireSubscriptionCreatedMessage::class,
        CreateAdditionalPendingOrders::class,
        GenerateReferralCode::class,   // …and more
    ],
];
```

In this repo: Real map: `src/app/Subscriptions/Providers/EventServiceProvider.php` (`SubscriptionCreated` fans out to 7 listeners). Listeners live in `Subscriptions/Listeners/`. The `VATServiceProvider` even carries a `TODO HANDLE EVENT LISTENERS` note.

**How it connects →** A special, built-in event source is an Eloquent model being saved or deleted — handled by observers ↓.

---

### 16. Model Observers
*listeners for a model's lifecycle*

**One-liner:** An observer is a listener dedicated to one Eloquent model — methods that run automatically when that model is created, updated or deleted.

**Plain English**

Eloquent fires events on every `save`/`delete`. An **observer** bundles the handlers for one model into a single class: `created()`, `updated()`, `deleted()`.

This keeps side-effects (busting a cache, writing a log, firing a message) *out* of the model and controller — they just happen whenever the data changes, wherever it changes.

**Analogy**

**A CCTV operator watching one room.** Whenever someone enters, leaves or moves something (the model changes), the operator reacts — so the room itself doesn't have to watch over its own shoulder.

**Real code**

```php
class BoxObserver {
    public function created(Box $box) { /* react to a new box */ }
    public function updated(Box $box) { /* … */ }
    public function deleted(Box $box) { /* … */ }
}
```

In this repo: `src/app/Observers/BoxObserver.php` (has `created/updated/deleted`), registered in `AppServiceProvider` via `Box::observe(BoxObserver::class)`.

**How it connects →** Some reactions are slow or external — push those off the request thread as background jobs ↓.

---

### 17. Jobs & Queues
*work done later, in the background*

**One-liner:** A job is a unit of work that can be queued and run later by a separate worker, so the web request returns immediately instead of waiting.

**Plain English**

Slow work (emails, external API calls, heavy processing) shouldn't block the user's request. A **job** that `implements ShouldQueue` is serialised onto a **queue**; a background **worker** picks it up and runs its `handle()` later — retrying if it fails.

**Analogy**

**The kitchen ticket rail.** The waiter (the request) doesn't cook at your table — they clip a ticket (the job) to the rail and move on. A chef (the queue worker) makes it when free. The waiter is freed up instantly.

**Real code**

```php
class CreateOrder implements ShouldQueue {
    public function handle(SubscriptionHandler $handler): bool {
        // runs later, on a worker — not during the web request
    }
}
```

In this repo: `src/app/Subscriptions/Packages/Subscription/Jobs/CreateOrder.php`. Note: the Pricing-move PR specifically scanned for `ShouldQueue` references — renaming a queued class can break jobs already serialised on the queue, because the class name is stored in the payload.

**How it connects →** That last point — class names getting persisted — is why relocations also check queues and morph-maps. Next: how the data itself is modelled ↓.

---

## Data & config

### 18. Eloquent & Models (ORM)
*database as objects*

**One-liner:** An ORM lets you work with database rows as PHP objects. A Model class maps to a table; its instances are rows.

**Plain English**

**ORM** = Object-Relational Mapper. Instead of writing SQL, you use objects. A **Model** (Laravel's Eloquent) represents one table: `Box` ↔ the `boxes` table, and `$box->num_recipes` reads a column.

Models also express **relationships** (a `Tariff` has many `boxes`), which the ORM turns into the right queries for you.

**Analogy**

**A translator between two languages.** Your code speaks "objects and properties"; the database speaks "tables and rows and SQL". The ORM translates each way so you can stay in object-land.

**Real code**

```php
// Seen in the GoustoTariffService test in this PR:
$box = new Box();
$box->num_recipes  = 3;     // a column on the boxes table
$box->num_portions = 2;
$tariff->boxes();             // a relationship → "the boxes for this tariff"
```

In this repo: Models live under `src/app/Models/` (e.g. `Box`, `Tariff`) and some domains keep their own, e.g. `App\BoxOrdering\Models\Order`.

**How it connects →** For models to have tables to map to, the schema must be created — migrations ↓.

---

### 19. Repositories
*one place to fetch/store data*

**One-liner:** A repository is the single class that knows how to load and save a kind of model, hiding the query details behind an interface.

**Plain English**

Instead of sprinkling Eloquent queries across controllers and services, code asks a **repository**: "give me the boxes for this tariff." The repository owns the query logic.

It usually sits behind an *interface* (a contract), so the data source can be swapped or faked in tests. It's **Eloquent + Contracts + DI** combined into one tidy pattern.

**Analogy**

**A librarian.** You ask for "the book on X." You don't care whether they walk to shelf 3 or order it in — the how-to-retrieve lives with them, not with you.

**Real code**

```php
interface BoxRepositoryInterface {
    public function find(int $id): Box;
    public function all(): Collection;
}
// injected by type-hint; the concrete impl is bound in a provider
```

In this repo: Interfaces in `src/app/Models/Interfaces/` (`BoxRepositoryInterface`, `CampaignRepositoryInterface`, …). The pricing graph pulls one via `app(BoxRepositoryInterface::class)`.

**How it connects →** Repositories read and write database tables — and those tables are themselves defined by migrations ↓.

---

### 20. Migrations
*version-controlled schema*

**One-liner:** Migrations are PHP files that describe database changes (create table, add column) so the schema is reproducible and shared.

**Plain English**

A **migration** is a script that changes the database structure — and is checked into git like code. Run them and every environment (your machine, CI, production) ends up with the same schema.

This is why you don't hand-edit production tables: you write a migration so the change is reviewable, repeatable and reversible.

**Analogy**

**Git, but for your database shape.** Instead of someone secretly rearranging the warehouse, every change is a logged, replayable instruction everyone runs in order.

**Real code**

```php
public function up() {
    Schema::create('boxes', function (Blueprint $t) {
        $t->id();
        $t->integer('num_recipes');
        $t->integer('num_portions');
    });
}
```

In this repo: This repo keeps SQL/migration state under `src/database/`; schema changes ship as migrations rather than manual DB edits.

**How it connects →** Schema, services and behaviour all vary by environment — that's config ↓.

---

### 21. Config & Environment
*settings without code changes*

**One-liner:** Config files hold settings; `.env` holds per-environment values (keys, DB creds) so the same code runs everywhere.

**Plain English**

**Config** files (in `config/`) are arrays of settings your code reads via `config('app.providers')`. **Environment** values (`.env`) are the bits that differ per machine — database host, API keys — kept out of code and out of git.

`config/app.php` is special: it lists every *service provider* and *facade alias*, which is why the relocation PR had to touch it.

**Analogy**

**The settings menu vs. the source code.** You change the volume in settings, not by re-soldering the speaker. Config lets you change behaviour without editing logic; `.env` is the per-device settings that shouldn't be shared.

**Real code**

```php
// config/app.php — the wiring index this PR edited
'providers' => [
    App\Pricing\BoundedContexts\Infrastructure\PricingServiceProvider::class,
    App\Pricing\BoundedContexts\Infrastructure\VAT\VATServiceProvider::class,
],
'aliases' => [ 'VAT' => …\VATFacade::class ],
```

In this repo: The three edits in `src/config/app.php` (two providers + the `VAT` facade alias) were pure namespace repoints — same keys, new addresses.

**How it connects →** Finally: how this specific codebase organises all of the above ↓.

---

## Patterns & practice

### 22. Service / Application Classes
*where business logic lives*

**One-liner:** Plain classes that hold the real business logic, sitting between thin controllers and the data layer.

**Plain English**

A controller should be thin — it shouldn't calculate prices itself. It hands off to a **service** like `PricingService`, which orchestrates models, repositories and other services to do the actual work.

Services are built by the container via DI, so they're easy to test and reusable from both HTTP and CLI. In this repo's DDD layout they often live in an `Application/` layer.

**Analogy**

**Hospital specialists.** Reception (the controller) doesn't perform surgery — it refers you to the surgeon (the service) who does the skilled work. Reception just directs traffic.

**Real code**

```php
// Thin controller delegates to a service:
$breakdown = $this->pricingService->getPriceBreakdown($order);
```

In this repo: The Pricing domain is full of these: `App\Pricing\BoundedContexts\Application\` (`PricingV2`, `OrderPricing`, `BoxPricing`) and `…\Infrastructure\Services\`, all wired in `PricingServiceProvider`.

**How it connects →** Services move data around — and in Laravel that data usually travels as Collections ↓.

---

### 23. Collections
*arrays with superpowers*

**One-liner:** Collections are Laravel's wrapper around arrays, giving you fluent, chainable methods like `map`, `filter` and `where` instead of manual loops.

**Plain English**

Eloquent queries and many APIs return a **Collection**, not a raw array. You transform data by chaining readable steps rather than writing `foreach` loops with temporary variables.

**Analogy**

**A conveyor belt with stations.** Each method — `filter`, `map`, `where` — is a station that transforms items as they roll past. Far clearer than hand-stacking everything in one loop.

**Real code**

```php
// From the GoustoTariffService test in this very PR:
collect($boxes)
    ->filter(fn(Box $b) => $b->num_recipes > 0)
    ->values();   // re-index the filtered collection
```

In this repo: `GoustoTariffServiceTest` (moved in this PR) uses `collect()`, `filter` and `where` over `Box` collections.

**How it connects →** Readable, well-structured code is only trustworthy once it's verified — that's testing ↓.

---

### 24. Testing (PHPUnit & Codeception)
*prove it works*

**One-liner:** Automated checks that your code behaves as intended — fast isolated unit tests, plus slower tests that boot the app or hit the API.

**Plain English**

Two tools here: **PHPUnit** for fast, isolated unit tests, and **Codeception** suites for slower tests that boot Laravel or call the real API.

Tests lean on DI: you swap real dependencies for **mocks**/fakes so you can test one class in isolation. This repo groups suites *by domain* — e.g. `pricing-api` and `pricing-unit-slow`.

**Analogy**

**A test kitchen.** Before a dish reaches the menu (production), you cook it under controlled conditions to confirm it comes out right every time.

**Real code**

```php
$tariff = $this->createMock(Tariff::class);   // a fake dependency
$result = $subject->getTariffPrices($tariff);
$this->assertCount(5, $result->prices);     // expectation
```

In this repo: Pricing tests now live under `src/tests/Pricing/Phpunit/` and `src/tests/Pricing/Codeception/`; the suites are registered in `codeception.yml` and `phpunit.xml` — all updated by this PR.

**How it connects →** Wired, tested code is finally organised by business area — domains ↓.

---

## This codebase's shape

### 25. Bounded Contexts / Domains (this repo's structure)
*how Gousto organises code*

**One-liner:** Code is grouped by business area (Pricing, BoxOrdering, Billing…) instead of by technical type, with rules about who may depend on whom.

**Plain English**

Rather than one giant `app/` with all controllers in one folder, this repo groups code by **domain**: everything Pricing-related lives under `app/Pricing/`. That's literally what PR #10301 was doing — moving Pricing into its own domain folder.

A **bounded context** is a stronger version of that idea from Domain-Driven Design: a self-contained area with its own model and clear boundaries. Crossing a boundary (e.g. BoxOrdering using a Pricing class) is allowed but tracked — that's why the PR talked about "boundary-crossing" classes and an alias shim.

`architecture/domains.php` is the official list of recognised domains; adding `'Pricing'` there is what makes the tooling treat `App\Pricing\…` as a proper, "placed" domain.

**Analogy**

**Departments in a company.** Pricing, Ordering and Billing each have their own office and own staff. They can request things from each other, but through clear doorways — not by wandering into each other's filing cabinets.

**Real code**

```php
// src/architecture/domains.php — the list of recognised domains
return [
    'OrderIncentivisation',
    'Pricing',        // ← added by this PR
    'Billing',
    // …
];
```

In this repo: The PR added `'Pricing'` to `domains.php`, moved 120+ files under `src/app/Pricing/`, and registered ownership in `CODEOWNERS` under a `# DOMAIN Pricing` block. PHPStan rules enforce the boundaries.

**How it connects →** That's the full stack: the framework runs the show; the container + providers + contracts wire your services; facades give shortcuts; routes/controllers/middleware/commands are the entry points; events/listeners/observers/jobs let things react and run in the background; models/repositories/migrations/config manage data; and domains organise it all by business area.
