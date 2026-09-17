---
name: laravel-patterns
description: Laravel framework best practices - Eloquent, service/repository patterns, form requests, API resources, queues, and testing conventions
---

# Laravel Development Patterns

Framework-specific patterns and conventions for building maintainable Laravel applications.

## When to Activate

Use this skill when working in a Laravel project (presence of `artisan`, `composer.json` with `laravel/framework`, or `app/Http/Controllers`). Applies to controllers, models, requests, jobs, and tests.

## Controllers

- Keep controllers thin — one action per method, no business logic inline.
- Prefer **single-action controllers** (`__invoke`) for endpoints that don't share related actions.
- Validate input via **Form Requests**, never inline `$request->validate()` in non-trivial controllers.
- Return **API Resources** (`JsonResource`) for API responses, never raw models or arrays.

```php
class StoreOrderController extends Controller
{
    public function __invoke(StoreOrderRequest $request, OrderService $orders): OrderResource
    {
        $order = $orders->create($request->validated());

        return new OrderResource($order);
    }
}
```

## Service Layer

- Business logic belongs in a **Service class**, not the controller or the model.
- Services are injected via the container — type-hint them in the constructor or method signature.
- One service per bounded concern (`OrderService`, `InvoiceService`), not one giant `AppService`.
- Services should be framework-agnostic where possible — no `request()` helper calls inside them; pass data in explicitly.

## Eloquent Patterns

- Avoid N+1 queries — always eager load with `with()` / `load()` when a relation will be accessed in a loop or a resource.
- Use **query scopes** for reusable query logic instead of repeating `where()` chains.
- Use **accessors/mutators** (`Attribute::make()`) for derived fields, not logic scattered in views or controllers.
- Prefer **mass assignment with `$fillable`** over `$guarded = []`.
- Wrap multi-step writes in `DB::transaction()`.

```php
class Order extends Model
{
    protected $fillable = ['customer_id', 'status', 'total'];

    public function scopePending(Builder $query): Builder
    {
        return $query->where('status', 'pending');
    }

    protected function total(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => $value / 100,
        );
    }
}
```

## Repository Pattern (when to use it)

Only introduce a repository layer when:
- The same complex query logic is reused across multiple services, **or**
- You need to swap data sources in tests (rare in most Laravel apps — prefer model factories + real DB in tests instead).

Don't wrap Eloquent in a repository "just because" — Eloquent models already are the repository pattern for most apps. Over-abstracting adds indirection without benefit.

## Validation

- One **Form Request** class per action (`StoreOrderRequest`, `UpdateOrderRequest`).
- Put authorization checks in `authorize()`, not in the controller.
- Use `Rule::` objects for anything beyond basic rules (unique-with-exceptions, enum checks, conditional rules).

## API Resources

- Every API response goes through a `JsonResource` or `ResourceCollection`.
- Keep resources declarative — no queries inside `toArray()`; load relations beforehand.
- Use `whenLoaded()` to avoid triggering lazy loads from the resource itself.

## Jobs & Queues

- Anything that calls an external API, sends email/SMS, or does non-trivial processing should be a **queued Job**, not inline in the request cycle.
- Jobs should be idempotent — re-running a job with the same input should not duplicate side effects.
- Set `$tries`, `backoff()`, and `failed()` explicitly rather than relying on defaults.

## Testing (Pest / PHPUnit)

- Use **model factories** for test data, never hand-built arrays.
- Feature tests hit routes (`$this->postJson(...)`) and assert on response + DB state (`assertDatabaseHas`).
- Unit tests target services and single methods in isolation — mock external dependencies (HTTP clients, mailers), not the database.
- Target meaningful coverage on services and Eloquent scopes, not 100% coverage on getters/setters.

```php
it('marks an order as paid', function () {
    $order = Order::factory()->pending()->create();

    app(OrderService::class)->markPaid($order);

    expect($order->fresh()->status)->toBe('paid');
});
```

## Migrations

- One logical change per migration — don't bundle unrelated schema changes.
- Always define foreign keys with `constrained()` and an explicit `onDelete()` policy.
- Never edit a migration that has already run in a shared environment — write a new one.

## Security Checklist

- Mass assignment: `$fillable` set on every model, never `$guarded = []` in production code.
- Authorization: every protected endpoint enforces a Policy or Gate through a Form Request, controller, or authorization middleware. Authentication middleware alone is not authorization.
- Raw queries: parameterize with bindings, never string-interpolate user input into `DB::raw()` or `whereRaw()`.
- Secrets: config values pulled from `.env` via `config()`, never `env()` outside config files.