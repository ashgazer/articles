---
tags:
  - machine-learning
  - route-handling
  - laravel-routing
  - controller-methods
  - routing-concepts
---

# Laravel: What Can Be Hooked Up to a Route?

A controller is the *most common* thing, but a route's action can be several different things. The key idea: at bottom, a route action only needs to resolve to **something callable**. "Controller method," "invokable controller," and "closure" are just the common *shapes* of that one idea. The rest (`view`, `redirect`, `resource`, `fallback`, `singleton`) are convenience helpers Laravel wraps around routing so you don't have to write a closure for common patterns.

So it's less "a fixed menu" and more "callables, plus a handful of shortcuts."

## Quick reference table

| Type | What it is | Use case |
|---|---|---|
| **Controller method** `[Ctrl::class, 'method']` | Points at one method on a controller class | The default for anything with real logic; keeps routes thin |
| **Invokable controller** `Ctrl::class` | A controller with a single `__invoke()` | One-action controllers — a route that does exactly one thing |
| **Closure** `fn () => ...` | Inline anonymous function | Tiny routes, quick prototypes, health checks (`/ping`) |
| **`Route::view()`** | Renders a Blade view directly | Static-ish pages (about, terms) with no logic, maybe some passed data |
| **`Route::redirect()` / `permanentRedirect()`** | Sends the user elsewhere (302 / 301) | Moved/renamed URLs, short-links, retiring old paths |
| **`Route::resource()` / `apiResource()`** | Registers a *set* of RESTful routes to one controller | Standard CRUD resources; `apiResource` drops the form routes for APIs |
| **`Route::singleton()`** | Resource routes for a resource with no ID | Single-instance resources (a profile, current subscription) |
| **`Route::fallback()`** | Catches requests matching no other route | Custom 404s, catch-all/SPA handling |
| **Any callable** (invokable object, `[$obj, 'method']`) | The underlying primitive the above build on | Rare directly; handing the router a pre-built callable |
| **Legacy string** `'Ctrl@method'` | Old string reference to a controller method | Only in older/legacy codebases — recognise, don't reach for |
| **Livewire full-page component** `Component::class` | A Livewire component as the whole page | Interactive pages when the team uses Livewire (looks like an invokable controller but isn't) |

## Examples

### 1. Controller method — the classic array syntax

```php
Route::get('/users', [UserController::class, 'index']);
```

### 2. Invokable controller — single `__invoke()`, pass just the class

```php
Route::get('/report', GenerateReport::class);
```

### 3. Closure — inline logic, no controller

```php
Route::get('/ping', fn () => 'pong');
```

### 4. A view, directly — render a template with no logic

```php
Route::view('/about', 'pages.about', ['team' => $team]);
```

### 5. A redirect — no handler, just send them elsewhere

```php
Route::redirect('/old', '/new');
Route::permanentRedirect('/legacy', '/home'); // 301
```

### 6. Resource controller — a whole set of RESTful routes at once

Maps `index`, `create`, `store`, `show`, `edit`, `update`, `destroy` to methods on one controller.

```php
Route::resource('photos', PhotoController::class);
Route::apiResource('photos', PhotoController::class); // no create/edit form routes
```

### 7. Singleton — resource routes with no ID

```php
Route::singleton('profile', ProfileController::class);
```

### 8. Fallback — catches anything unmatched

```php
Route::fallback(fn () => response('Not found', 404));
```

### 9. Livewire full-page component

Looks identical to an invokable controller, but it's Livewire handling it, not a plain `__invoke`.

```php
Route::get('/dashboard', Dashboard::class); // Dashboard is a Livewire component
```

### Legacy string syntax (recognise, don't reach for)

The pre-Laravel-8 style. Removed as the automatic default, but common in older code.

```php
Route::get('/users', 'UserController@index');
```

## Handler vs. what you attach to a route

Important distinction: everything above is what **handles** a route. Separately, there are things you **attach** to a route that aren't the handler:

- **Middleware** — `->middleware('auth')`
- **Names** — `->name('users.index')`
- **Route model binding** — type-hint a model in the handler so Laravel auto-hydrates it from the URL
- **Groups / prefixes / domains** — wrap multiple routes with shared config

These decorate or wrap the route rather than being the thing that answers it.

## One thing outside this list entirely

**Laravel Folio**, if your team uses it, is page/file-based routing where a Blade file's *location* defines the route — so there's no `Route::` line at all. Different paradigm; worth knowing the name exists.
