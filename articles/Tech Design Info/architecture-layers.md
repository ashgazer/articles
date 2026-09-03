---
tags:
  - machine-learning
  - bounded-context
  - clean-architecture
  - domain-driven-design
  - microservices
---

# The Layers of Gousto Core, Explained Simply 🧅

A request comes in. It travels through a few layers, each with one job, and an answer travels back out. This page walks you through every layer using **real code from this repo** (the `Address` flow in the `Subscriptions` domain).

## The one idea to hold onto

Each layer should do **one kind of job** and hand off to the next. The whole point is: when you need to change something, there's an **obvious place** for that change to live — and you don't accidentally break unrelated things.

If you mix jobs (e.g. database queries inside a controller, or HTTP stuff inside a model), the code gets tangled and scary. Keeping jobs separate is the entire game. Click a layer below.

## A request's journey →

The main request pipeline flows through these layers, in order:

1. **Route** — the address book
2. **Controller** — the traffic cop
3. **Service / BoundedContext** — the brain
4. **Repository** — the librarian
5. **Model** — the thing itself

⟵ And the response travels back out: **Model → Presenter → Controller → JSON to the customer**

**Legend (all layers covered on this page):** Route, Controller, Service / BoundedContext, Repository, Model, Presenter, Interface / Contract

## Layer details

### Route
*the address book*

> "When a URL like this is hit, call that method."

A **Route** is just a lookup table. It maps a URL + HTTP verb (GET, POST…) to one controller method. It contains no logic — it just points.

**File:** `src/routes/api.php`

- **Do:**
  - Map a URL to **one** controller method
  - Group routes & attach middleware (auth, throttling)
  - Stay tiny and declarative
- **Don't:**
  - Contain business logic
  - Query the database
  - Make decisions about data

```php
// routes/api.php — just a pointer, no logic
Route::get(
  'users/{user}/addresses/{address_id}',
  [AddressController::class, 'show']
);
```

### Controller
*the traffic cop*

> "Take the HTTP request, call the right helpers, return an HTTP response."

A **Controller** is the bridge between the web and your app. It reads the request, asks other layers to do the real work, and shapes the reply. It should be **thin** — a coordinator, not a doer.

**File:** `src/app/Subscriptions/Http/Controllers/AddressController.php`

- **Do:**
  - Read input from the request
  - Call repositories / services / bounded contexts
  - Catch exceptions → turn into HTTP error responses
  - Return `apiSuccess(...)` / a presenter
- **Don't:**
  - Write SQL or complex queries (→ Repository)
  - Hold business rules (→ Service / BoundedContext)
  - Format the JSON shape by hand (→ Presenter)

```php
class AddressController extends BaseController {
  // dependencies are injected — the controller just *uses* them
  public function __construct(
    AccountManagement $accountManagement,
    AddressRepositoryInterface $addresses,
    UserRepositoryInterface $users,
  ) { /* ...store them... */ }

  public function show($user, $address_id) {
    try {
      // ask the repository to fetch — don't query here yourself
      $address = $this->addresses->getBy([
        'user_id' => $user->id,
        'id'      => $address_id,
      ])->first();

      if (!$address) {
        throw new NotFoundModelException('Unable to determine Address');
      }
    } catch (\Exception $e) {
      return $this->apiException($e); // → HTTP error
    }
    return self::apiSuccess(['address' => $address->toArray()]);
  }
}
```

### Service / BoundedContext
*the brain*

> "The business rules and decisions live here — no HTTP, no SQL."

This is where the **actual logic of the business** lives: "can this user add an address?", "what happens to their delivery slot when they move house?". In this repo the clean entry point to a domain is a **BoundedContext** interface (e.g. `AccountManagement`); older code calls these "Services". Both mean the same thing: **the brain**.

**File:** `src/app/Subscriptions/BoundedContexts/AccountManagement.php`

- **Do:**
  - Hold business rules & decisions
  - Orchestrate several repositories/models for one task
  - Throw meaningful domain exceptions
  - Be callable from a controller *or* a CLI command, identically
- **Don't:**
  - Know it was triggered by HTTP
  - Build raw queries (delegate to a Repository)
  - Format JSON for the API (→ Presenter)

```php
// A BoundedContext = the declared "front door" to a domain's logic.
// The controller depends on this interface, not the guts behind it.
interface AccountManagement {
  public function getUserById(int $id): ?UserDetails;

  /** @throws InvalidUserShippingAddressException */
  public function getShippingAddressForUser(int $userId): AddressDetails;

  // "create an address, applying all the rules" — the real work,
  // with zero knowledge of HTTP or JSON.
  public function newAddressFor(array $attrs): Address;
}
```

### Repository
*the librarian*

> "All database reads/writes for a model go through me."

A **Repository** is the only thing that talks to the database for a given model. Need addresses matching some filters? Ask the repository. This keeps query logic in one place and lets other layers stay ignorant of SQL.

**File:** `src/app/Subscriptions/Models/Repositories/AddressRepository.php`

- **Do:**
  - Build & run queries (filters, sorting, pagination)
  - Return models or collections
  - Hide *how* data is fetched behind a method name
- **Don't:**
  - Make business decisions (→ Service)
  - Format output for the API (→ Presenter)
  - Know about HTTP requests

```php
class AddressRepository extends EloquentBaseRepository
    implements AddressRepositoryInterface {

  protected $model_name = Address::class;

  // "give me addresses matching these filters" — query logic lives HERE,
  // not scattered across controllers.
  public function get($filters = []) {
    $query = $this->getNewInstance()
                  ->with(['user', 'orders']);
    $this->applyFilters($query, $filters);
    $query = $this->order($query);
    return $this->paginate($query);
  }
}
```

### Model
*the thing itself*

> "I represent one row + the rules about what makes me valid."

A **Model** (an Eloquent model) maps to a database table — one instance = one row. It knows its own fields, its relationships ("an Address belongs to a User"), and small rules about itself (validation, casting). It is the **noun** your app is about.

**File:** `src/app/Subscriptions/Models/Address.php`

- **Do:**
  - Define columns, casts & relationships
  - Hold validation rules for itself
  - Expose scopes (e.g. `->pending()`)
- **Don't:**
  - Run application workflows (→ Service)
  - Reach out to HTTP/APIs
  - Decide how it's shown to the user (→ Presenter)

```php
/**
 * @property int    $id
 * @property string $postcode
 * @property int    $user_id
 */
class Address extends AbstractSearchableModel implements Validable {
  use SoftDeletes, ValidatesModel;

  // a relationship: this Address belongs to a Country
  public function country() {
    return $this->belongsTo(Country::class);
  }
  // the data + its own rules — nothing about HTTP or JSON shape
}
```

### Presenter
*the gift-wrapper*

> "Take a model and turn it into the exact JSON the API promised."

A **Presenter** decides the **shape of the response**. The model has lots of fields; the presenter picks which ones go out, renames them, and formats them. This means changing the API's JSON never means touching your database code.

**File:** `src/app/Subscriptions/Presenters/AddressPresenter.php`

- **Do:**
  - Pick & rename fields for the API
  - Format values (booleans, dates, nesting)
  - Keep the public JSON contract in one place
- **Don't:**
  - Fetch data (it receives an already-loaded model)
  - Contain business rules
  - Decide HTTP status codes (→ Controller)

```php
class AddressPresenter extends AbstractPresenter {
  // turn the rich model into the precise JSON the API promised
  public function render($relations = [], $data = []) {
    $data = [
      'id'       => $this->id(),
      'line1'    => $this->string('line1'),
      'town'     => $this->string('town'),
      'postcode' => $this->string('postcode'),
      'default'  => $this->boolean('default'),
      // note: NOT every column — only what the customer should see
    ];
    return parent::render($relations, $data);
  }
}
```

### Interface / Contract
*the promise*

> "A list of methods a class promises to provide — so layers depend on a promise, not a thing."

Notice the controller asked for `AddressRepositoryInterface`, not `AddressRepository`. An **Interface** is just a **promise** ("anything I give you will have a `getBy()` method"). This lets you swap the real repository for a fake one in tests, and is the boundary the repo's **architecture tests** enforce (see ADR 0004).

**File:** `src/app/Subscriptions/Models/Interfaces/AddressRepositoryInterface.php`

- **Do:**
  - Define the methods a layer can be called with
  - Let you inject real *or* test versions
  - Mark the official "front door" of a domain
- **Don't:**
  - Contain any actual logic (no method bodies)
  - Touch the database or HTTP

```php
// A promise, not an implementation. The controller trusts this.
interface AddressRepositoryInterface {
  public function getBy(array $filters, array $relations = []);
  public function get($filters = []);
}
// In production Laravel binds this → real AddressRepository.
// In a test you can bind it → a fake that returns canned data.
// The controller never knows the difference. That's the point.
```

## 🍽️ If it helps: think of a restaurant

Same idea, no code. Everyone has one job and passes the order along.

| Layer | Role | Description |
|---|---|---|
| Route | The door + host | Sends you to the right table. Doesn't cook. |
| Controller | The waiter | Takes your order, brings it to the kitchen, returns with the plate. Cooks nothing. |
| Service | The head chef | Decides how the dish is made — the actual know-how. |
| Repository | The pantry keeper | The only one who goes into the storeroom to fetch ingredients. |
| Model | The ingredient | A tomato. Knows it's a tomato. Doesn't cook itself. |
| Presenter | Plating | Arranges the food beautifully before it leaves the kitchen. |

## 👣 Follow one real request: `GET /users/42/addresses/7`

This is the actual `AddressController::show()` flow in this repo.

1. **Request hits a Route** (Layer: Route) — `routes/api.php`
   `GET /users/42/addresses/7` matches a route, which points at `AddressController@show`.
2. **Controller takes over** (Layer: Controller) — `AddressController::show()`
   It reads `$user` and `$address_id` from the URL. It does NOT query the DB itself.
3. **Controller asks an injected repository** (Layer: Interface / Contract) — `AddressRepositoryInterface`
   It calls `$this->addresses->getBy([...])` — talking to a promise, not a concrete class.
4. **Repository runs the query** (Layer: Repository) — `AddressRepository::getBy()`
   This is the only layer that touches the database. It builds the query and returns a model.
5. **A Model comes back** (Layer: Model) — `Address` (one row)
   An `Address` object representing that database row, with its relationships available.
6. **Controller handles the "not found" case** (Layer: Controller) — `throw NotFoundModelException`
   If nothing came back, it throws — which becomes a clean HTTP 404, not a crash.
7. **Presenter shapes the JSON** (Layer: Presenter) — `AddressPresenter::render()`
   (For richer endpoints) the model is wrapped so only the agreed fields go out, correctly formatted.
8. **Controller returns the response** (Layer: Controller) — `self::apiSuccess([...])`
   The shaped data is wrapped in the standard API envelope and sent back to the customer as JSON.

## 📋 Best-practice scorecard — what we have, partly have, and are missing

These are the layering & Domain-Driven-Design rules teams aim for. Each one is marked by where **this repo actually stands today**, with real evidence. Good news: you already do more of these than you'd think (DTOs included!).

**Summary:** 7 practices you already follow · 3 partially there — easy wins · 2 genuine gaps to consider

### ✓ HAVE — Pass DTOs between layers — never raw DB models
*Category: DDD*

**What it means:** A DTO (Data Transfer Object) is a plain, read-only bundle of fields. Instead of handing an Eloquent model around, you hand a DTO. The model is a live DB row (mutable, lazy-loads more queries when you touch a relation); a DTO is a frozen snapshot of just the data this layer needs.

**Why it matters:** Eloquent models leak the database shape into your domain and HTTP edges. If a column is renamed, everything that touched the model breaks. A DTO is a stable contract between layers — and being readonly, nobody can mutate it by accident.

**Where we stand — in this repo:** You already do this. `AddressDetails` and `UserDetails` are `readonly class`es with a `::from(Address)` factory, and the `AccountManagement` BoundedContext returns *those*, not the model. That is the pattern, done right.

**Look here:** `src/app/Subscriptions/BoundedContexts/AddressDetails.php`

**Warning smell:** A controller or service receiving an Eloquent model and reading `$model->some_column` directly — the DB shape has escaped its cage.

### ✓ HAVE — Depend on interfaces, not concrete classes (Dependency Inversion)
*Category: SOLID*

**What it means:** High-level code (controllers, services) should ask for an interface — a promise of methods — not a specific class. Laravel then "injects" the real implementation at runtime.

**Why it matters:** You can swap the real class for a fake in tests, and a domain gets a clear, swappable "front door". Code stops being welded to one implementation.

**Where we stand — in this repo:** The controller constructor asks for `AddressRepositoryInterface`, `UserRepositoryInterface`, and the `AccountManagement` interface — never the concrete repository.

**Look here:** `src/app/Subscriptions/Http/Controllers/AddressController.php`

**Warning smell:** `new SomeRepository()` or `new SomeService()` inside a controller — you have hard-wired a concretion and made it un-testable.

### ✓ HAVE — Enforce architecture rules with automated tests
*Category: Process*

**What it means:** Instead of trusting people to remember the rules, write tests that fail the build when a boundary is crossed (e.g. a controller reaching into another domain's internals).

**Why it matters:** Prose rules drift; the moment they are not enforced, they rot. A failing test is an un-ignorable rule. This is genuinely advanced — most codebases never get here.

**Where we stand — in this repo:** You have this: PHPat rules run alongside PHPStan to enforce that `App\<Domain>\` internals are only reachable through a declared surface.

**Look here:** `docs/adr/0004-architectural-tests.md` · `src/architecture/README.md`

**Warning smell:** "We have a rule about X" with nothing checking it — assume it is already being violated somewhere.

### ✓ HAVE — One root per domain (bounded contexts)
*Category: DDD*

**What it means:** All code for a domain lives under one folder/namespace (`src/app/<Domain>/`), rather than being scattered across global `Services/`, `Models/` piles.

**Why it matters:** You can see a whole domain in one place, and the boundaries between domains become explicit and defensible instead of accidental.

**Where we stand — in this repo:** Codified in your guidelines: domain-bound files live under a single PSR-4 root with mirrored test/factory trees.

**Look here:** `docs/adr/0002-how-to-scope-a-domain.md`

**Warning smell:** Adding a new file to a top-level `src/app/Services/` grab-bag instead of a domain folder.

### ✓ HAVE — Side effects via domain events, not inline code
*Category: DDD*

**What it means:** When something significant happens ("subscription created"), fire an event. Other concerns (email, analytics) subscribe via listeners, instead of the creating code calling them directly.

**Why it matters:** The core action stays focused on its job. You can add/remove reactions without touching the thing that triggered them.

**Where we stand — in this repo:** You have a family of these: `SubscriptionCreated`, `SubscriptionUpdated`, `SubscriptionReactivated` with listeners.

**Look here:** `src/app/Subscriptions/Events/Model/SubscriptionCreated.php`

**Warning smell:** A "create" method that also sends 3 emails, writes to analytics, and calls a third-party API inline — those should be listeners.

### ✓ HAVE — Explicit domain exceptions, not generic ones
*Category: DDD*

**What it means:** Throw meaningful, named exceptions (`UserDoesNotExistException`) instead of a bare `\Exception('not found')`.

**Why it matters:** Callers can catch the specific case they care about, and the error tells you exactly what business rule was violated.

**Where we stand — in this repo:** You do this widely — e.g. `UserDoesNotExistException`, `InvalidUserShippingAddressException`, `SubscriptionSlotUpdateException`.

**Look here:** `src/app/Subscriptions/BoundedContexts/UserDoesNotExistException.php`

**Warning smell:** `throw new \Exception("something went wrong")` — un-catchable in any useful way.

### ✓ HAVE — API contracts are first-class (OpenAPI)
*Category: Contracts*

**What it means:** The shape of every endpoint is described in a spec (OpenAPI/Swagger), treated as part of the architecture — not an afterthought.

**Why it matters:** Consumers know exactly what to expect; drift between code and docs is caught. Your guidelines call contracts "a critical part of the system's architecture".

**Where we stand — in this repo:** Endpoints carry `@OA` annotations (see the schema blocks in `AddressController`) and there is a dedicated `OpenApi` tree.

**Look here:** `src/app/OpenApi/` · `GUIDELINES.md` (Contracts are first-class)

**Warning smell:** Adding an endpoint with no spec, so nobody can tell what it returns without reading the code.

### ~ PARTIAL — Wrap domain concepts in Value Objects
*Category: DDD*

**What it means:** A Value Object is a tiny immutable type that wraps a primitive plus its rules — e.g. a `PostalCode` instance that can only ever hold a valid, formatted postcode, instead of passing a bare `string` around.

**Why it matters:** A postcode is never "just a string". A real VO makes invalid states unrepresentable: once you hold a PostalCode object, it is guaranteed valid. Bare strings let bad data travel deep before blowing up.

**Where we stand — in this repo:** Partial: you have `PostalCode`, but it is a class of *static helpers* (`::isValid()`, `::format()`) — useful, but you still pass raw strings around. A fuller VO would be an instance you construct once and trust thereafter.

**Look here:** `src/app/Subscriptions/Packages/Address/PostalCode.php`

**Warning smell:** `string $postcode` threaded through ten methods, each re-validating it — that validation wants to live inside a PostalCode type.

### ~ PARTIAL — Keep controllers thin — no domain logic, no ORM leakage
*Category: Layering*

**What it means:** A controller should read input, call one service/repository, and return a response. Business decisions and query-building belong elsewhere.

**Why it matters:** Thin controllers are easy to read and the logic underneath is reusable from a CLI command or a queue worker, not just HTTP.

**Where we stand — in this repo:** Mixed: newer flows delegate cleanly to BoundedContexts. But some older actions still do work inline — e.g. `AddressController::show()` calls `$address->toArray()` and reshapes `country` by hand, leaking the model past the controller instead of using a DTO/Presenter.

**Look here:** `src/app/Subscriptions/Http/Controllers/AddressController.php`

**Warning smell:** `->toArray()`, manual array reshaping, or "if this business case then…" branches inside a controller method.

### ~ PARTIAL — Rich models vs anemic models
*Category: Layering*

**What it means:** An "anemic" model is just data + getters, with all behaviour living in services. A "rich" model holds the behaviour that naturally belongs to it (e.g. `$order->cancel()`).

**Why it matters:** Classic DDD prefers rich models so rules live with the data they govern. But anemic-model + service is a deliberate, common, perfectly workable trade-off — the point is to choose consciously, not drift.

**Where we stand — in this repo:** Your models lean anemic (data + validation traits), with workflows in services/BoundedContexts. Not wrong — just know it is a trade-off, and keep truly model-local rules (e.g. validity) on the model.

**Look here:** `src/app/Subscriptions/Models/Address.php`

**Warning smell:** A service named `AddressManagerHelperUtil` doing everything, while the model is a hollow bag of fields — fine if intentional, a smell if accidental.

### ○ GAP — Validate input at the boundary (Form Requests)
*Category: Validation*

**What it means:** Laravel's Form Request classes validate and authorise an incoming request *before* the controller body runs, keeping validation rules in one declarative place.

**Why it matters:** Bad input should be rejected at the door, not discovered deep in a service via a half-built model. It also keeps validation rules out of controllers and models.

**Where we stand — in this repo:** Gap: there is no `Http/Requests/` tree. Input is validated later — via model-level traits (`ValidatesModel` on `Address`) and ad-hoc controller checks. That works, but mixes "is this request well-formed?" with "is this entity valid?", two different questions.

**Look here:** (absent) — compare: Laravel `app/Http/Requests/*FormRequest.php`

**Warning smell:** A controller manually doing `if (!Request::input('postcode')) throw …` at the top of several methods — that is a Form Request waiting to be born.

### ○ GAP — Never let the ORM cross the repository boundary
*Category: Layering*

**What it means:** Eloquent query builders, collections, and lazy relationships should stay inside repositories. Past that line, hand back DTOs or domain objects.

**Why it matters:** If Eloquent collections flow into controllers and views, the whole app silently depends on the ORM, and swapping data sources or reasoning about queries (N+1!) becomes guesswork.

**Where we stand — in this repo:** Gap (improving): DTOs like `AddressDetails` are exactly the fix and are used in newer code — but raw models and `->toArray()` still cross the line in older controllers. The DTO habit is the direction of travel.

**Look here:** `src/app/Subscriptions/Models/Repositories/AddressRepository.php`

**Warning smell:** An Eloquent `Collection` or model appearing as a controller return value or in a presenter that then lazy-loads relations.

## 🔧 Deep dive: fixing two gaps, on your real code

Concrete before/after for the two genuine gaps — using `AddressController::store()` and your `PostalCode`, as they exist today. This is the *migration* shape your guidelines describe ("new and changed code should move towards" the target), not a big-bang rewrite.

### Gap 1 · Form Requests — Validate at the boundary with a Form Request

Move the scattered "is this input OK?" checks out of the controller into one declarative class that runs before the controller body.

**✕ BEFORE — today**

```php
// AddressController::store() — TODAY
public function store(User $user, $type = 'shipping') {
  // ❌ validation tangled into the controller, repeated per endpoint
  if (!in_array($type, $this->allowable_address_types)) {
    return $this->apiError(self::UNKNOWN_ADDRESS_PURPOSE_MESSAGE);
  }
  if (!Request::exists('postcode')) {
    return $this->apiError(self::POSTCODE_IS_MISSING_MESSAGE);
  }
  // ...only NOW does the real work begin
}
```

**✓ AFTER — target**

```php
// 1) NEW: src/app/Subscriptions/Http/Requests/StoreAddressRequest.php
class StoreAddressRequest extends FormRequest {
  public function rules(): array {
    return [
      'type'     => ['required', Rule::in(['shipping','billing'])],
      'postcode' => ['required', new ValidPostcode($this->market())],
      'line1'    => ['required', 'string'],
      'town'     => ['required', 'string'],
    ];
  }
}

// 2) Controller shrinks — invalid input never reaches it (auto-422)
public function store(StoreAddressRequest $request, User $user) {
  $address = $this->accountManagement
                  ->newAddressFor($request->validated());
  return self::apiSuccess(['created' => AddressDetails::from($address)]);
}
```

**Steps:**

1. Laravel sees the typed `StoreAddressRequest` parameter and validates **before** the method body runs. Bad input becomes a clean `422` automatically — no manual `if`/`apiError`.
2. All rules for "what a valid create-address request looks like" live in **one place**, reusable and testable on their own.
3. The controller drops from a wall of guards to ~2 lines: it now only **coordinates**. And notice it returns a `AddressDetails` DTO, closing the `->toArray()` leak too.
4. It pairs with Gap 2: the `ValidPostcode` rule simply wraps the `PostalCode` value object (next tab).

**Migration note:** You are not deleting the old model-level `ValidatesModel` checks — those answer a different question ("is this **entity** valid to persist?"). The Form Request answers "is this **request** well-formed?". Keeping both, but separate, is the point.

### Gap 2 · Value Object — Turn PostalCode into a real Value Object

Today it is a bag of static helpers, so raw strings still travel around and every caller must remember to validate. Make it a type you construct once and trust forever.

**✕ BEFORE — today**

```php
// PostalCode — TODAY: static helpers over a raw string
class PostalCode {
  public static function isValid(string $pc, MarketCode $m): bool { /*...*/ }
  public static function format(string $pc, MarketCode $m): string { /*...*/ }
}

// callers pass bare strings + must REMEMBER to check, every time:
if (!PostalCode::isValid($input, $market)) { /* handle */ }
$stored = PostalCode::format($input, $market);
// nothing stops an UNVALIDATED string flowing on from here ⚠️
```

**✓ AFTER — target**

```php
// PostalCode — AFTER: an instance that cannot exist if invalid
final readonly class PostalCode {
  private function __construct(
    public string $value,
    public MarketCode $market,
  ) {}

  // the ONLY way in: validates + formats, or throws
  public static function fromString(string $raw, MarketCode $market): self {
    $formatted = self::applyMarketSpacing($raw, $market);
    if (!preg_match(self::REGEX_BY_MARKET[$market->value], $formatted)) {
      throw new InvalidPostcodeException($raw, $market);
    }
    return new self($formatted, $market);
  }

  public function __toString(): string { return $this->value; }
}

// once you hold one, it is GUARANTEED valid + formatted:
$postcode = PostalCode::fromString($raw, $market);
$address->postcode = (string) $postcode;
```

**Steps:**

1. **Private constructor + `fromString()` factory** = the only door in. You literally *cannot* hold a `PostalCode` that is invalid or unformatted — invalid states are unrepresentable.
2. The validate-then-format ritual happens **once**, inside the type. No caller can forget it.
3. `readonly` means it never changes after construction — safe to pass anywhere, including across layers as part of a DTO.
4. Your `StoreAddressRequest` rule from Gap 1 becomes trivial: `ValidPostcode` just tries `PostalCode::fromString()` and reports the thrown exception as a validation error.

**Migration note:** This is incremental: keep the static helpers working while you migrate callers to `fromString()` one at a time. The win compounds — every layer that accepts a `PostalCode` instead of a `string` gets the guarantee for free.

## ❓ Questions you're probably too embarrassed to ask (don't be)

**What's the difference between a Service and a Repository? They both feel like "helpers".**
A Repository only fetches/saves data (it talks to the database). A Service makes decisions and runs the business rules — it often calls one or more repositories to do its job. Rule of thumb: if it contains an "if" about how the business works, it's a Service; if it's "get/save these rows", it's a Repository.

**My controller is getting big. Where do I move code?**
Ask what KIND of code it is. Database query → Repository. A business rule or multi-step workflow → Service/BoundedContext. Deciding the JSON shape → Presenter. After moving those out, the controller should be a short coordinator: read input, call a thing, return a response.

**Why does the controller ask for an *Interface* instead of the real Repository?**
So you can swap the real one for a fake in tests, and so a domain has a clear "front door". The controller depends on a promise ("you will have a getBy method"), not on the concrete class. This repo even has architecture tests (ADR 0004) that fail the build if you reach past these boundaries.

**What is a "BoundedContext"? I only ever heard "Service".**
Same idea, newer name. It's the declared, public entry point into a domain's logic (e.g. AccountManagement for Subscriptions). Older code calls these "Services" and they live in a Services/ folder. New code in this repo prefers the BoundedContext pattern — see GUIDELINES.md and ADR 0002.

**Where does a brand-new feature's code go?**
Per GUIDELINES.md: under one domain root — `src/app/<Domain>/` (namespace `App\<Domain>`). Inside it you'll find the same layers shown here: `Http/Controllers`, `Services` or `BoundedContexts`, `Models`, `Models/Repositories`, `Presenters`. Put each piece in the folder that matches its job.

**Is it ever OK to break these rules?**
Yes — GUIDELINES.md literally says existing code diverges and that's known tech debt. The rule is "new and changed code should move towards these layers." Don't rewrite the world; just put the code you're writing today in the right place.

## Footer

Built from the real code in `src/app/Subscriptions/`. See `GUIDELINES.md` and `docs/adr/` for the official rules.

Layers are about **where code lives**, not how clever it is. When in doubt, ask: "what *kind* of job is this?"
