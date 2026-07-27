---
tags:
  - machine-learning
  - reflection-api
  - dependency-injection
  - laravel-features
  - php-reflection
---

# Laravel: What is `ReflectionMethod`?

In Laravel (and PHP generally), `ReflectionMethod` is a built-in PHP class from the Reflection API — not something Laravel invented, but Laravel uses it heavily under the hood.

## What it does

It lets you inspect a class method at runtime: its name, parameters, visibility, return type, attributes, doc comments, etc. — and even invoke it dynamically, including private/protected methods if needed.

```php
$reflection = new ReflectionMethod(SomeClass::class, 'someMethod');

$reflection->getName();         // 'someMethod'
$reflection->isPublic();        // true/false
$reflection->isStatic();        // true/false
$reflection->getParameters();   // array of ReflectionParameter
$reflection->getReturnType();   // ReflectionNamedType or null
$reflection->invoke($instance, ...$args); // call it dynamically
```

## Where Laravel uses it

The main place you'll bump into it is **dependency injection / the service container**. When Laravel resolves a controller method or a class method (e.g. via `app()->call()`), it uses `ReflectionMethod` to look at the method's parameters and figure out what type-hinted classes need to be auto-resolved and injected.

For example, this is roughly what happens behind the scenes when you type-hint a `Request` or a model in a controller method:

```php
public function show(Request $request, Post $post) { ... }
```

Laravel reflects on `show()`, sees the parameter types, and resolves each one out of the container (or route model binding) before calling the method.

You'll also see it in:
- **Event/listener resolution** — matching listener method signatures
- **Command bus / job handling** — figuring out `handle()` method dependencies
- **Testing/mocking utilities** — some packages use it to inspect or call protected/private methods
- **Artisan command argument binding**

## When you'd use it yourself

Usually only in advanced scenarios: building your own mini-container/DI logic, writing a package that needs to auto-wire method arguments, or in tests when you want to call a private/protected method directly:

```php
$method = new ReflectionMethod($object, 'protectedMethod');
$method->setAccessible(true); // not needed in PHP 8.1+, methods are accessible by default now
$result = $method->invoke($object, $arg1, $arg2);
```
