---
tags:
  - machine-learning
  - unit-testing
  - phpunit
  - laravel
  - testing-best-practices
---

# PHP / Laravel: What is a Data Provider?

In PHP/Laravel, "data provider" almost always means a **PHPUnit data provider** — a testing feature. Laravel doesn't have its own separate concept; it just uses PHPUnit (or Pest) underneath, and this comes from there.

## What it is

A data provider is a **method that supplies multiple sets of inputs to a single test**, so the test runs once per data set instead of you copy-pasting near-identical tests. It's how you check "does this behave correctly across many cases" without repetition (very DRY).

The provider returns an array (or yields) where each entry is one set of arguments passed into the test method:

```php
use PHPUnit\Framework\Attributes\DataProvider;

class DiscountTest extends TestCase
{
    #[DataProvider('discountCases')]
    public function test_discount_is_applied(int $total, int $expected): void
    {
        $this->assertEquals($expected, applyDiscount($total));
    }

    public static function discountCases(): array
    {
        return [
            'no discount'   => [50, 50],
            'ten percent'   => [100, 90],
            'bulk discount' => [1000, 800],
        ];
    }
}
```

That runs `test_discount_is_applied` three times, once per row, each with different `$total`/`$expected` values. The string keys (`'no discount'`, etc.) are optional labels that show up in the test output so you can see *which* case failed.

## Things worth knowing

- The provider method must be **`static`** and **`public`** in modern PHPUnit.
- It runs *before* the test setup — so you can't rely on things created in `setUp()` inside it.

## The two syntaxes (attribute vs annotation)

The `#[DataProvider('discountCases')]` line **is** a PHP attribute. Older code uses the doc-block form instead:

```php
/**
 * @dataProvider discountCases
 */
public function test_discount_is_applied(int $total, int $expected): void
```

If your team is on **PHPUnit 10+** they'll use the attribute; older projects use the annotation. Same behaviour either way.

## If your team uses Pest

The equivalent in Pest is the `->with([...])` dataset syntax, which does the same job with different wording:

```php
it('applies discount', function (int $total, int $expected) {
    expect(applyDiscount($total))->toBe($expected);
})->with([
    'no discount'   => [50, 50],
    'ten percent'   => [100, 90],
    'bulk discount' => [1000, 800],
]);
```
