---
name: doctrine
description: >-
  Idiomatic Doctrine ORM/DBAL on Symfony 7.x / PHP 8.3+ — entity mapping,
  associations and fetch strategy, repositories as the persistence boundary, the
  EntityManager/UnitOfWork model, querying, hydration, migrations, and the
  performance and memory traps (N+1, EAGER, flush-in-loop, unbounded identity
  map). Use when mapping entities, writing repositories or DQL/QueryBuilder,
  tuning fetch/hydration, writing migrations, or debugging Doctrine behavior.
  Triggers: "Doctrine", "entity", "repository", "DQL", "QueryBuilder",
  "association mapping", "migration", "EntityManager", "flush", "N+1",
  "hydration", "lazy loading".
---

# Doctrine

Doctrine is a **data-mapper** ORM: entities are plain objects mapped to rows, and
the `EntityManager` tracks the ones it knows about and writes their changes on
`flush()`. Most Doctrine pain comes from forgetting those two facts — that fetches
are lazy by default and that the manager holds everything it touches. This skill
is the idiomatic use plus those traps. It pairs with `database-design` (the schema)
and `performance` (query cost).

## Entities and mapping

- Map with attributes on 7.x. Entities model the **domain** — they hold state and
  enforce invariants in constructors and methods, not just public getters/setters.
- Avoid the anemic extreme (a bag of setters with no behavior) *and* the bloated
  extreme (persistence orchestration or formatting inside the entity). Domain
  logic that needs collaborators belongs in a service (`engineering-standards`).

## Associations and fetch strategy

The default fetch mode is **LAZY**, and that is correct — but it is also the source
of N+1. Load related data *explicitly* when you will use it:

```php
// N+1: lazy association loaded once per row while iterating
$orders = $repo->findAll();
foreach ($orders as $o) { $o->getCustomer()->getName(); }

// Fix: fetch-join loads customers with the orders in one query
$orders = $repo->createQueryBuilder('o')
    ->addSelect('c')
    ->join('o.customer', 'c')
    ->getQuery()->getResult();
```

- **Never set `fetch: EAGER` as a default** — it silently loads graphs on every
  query, including ones that do not need them (`database-design`, `performance`).
- Use `EXTRA_LAZY` for large collections you only count or paginate, so Doctrine
  issues `COUNT`/`LIMIT` instead of loading the whole collection.

## Repositories as the persistence boundary

Keep queries in repositories, not scattered through services and controllers. A
repository is the seam in front of the ORM (`architecture`, dependency direction):

```php
final class OrderRepository extends ServiceEntityRepository
{
    public function findOpenForCustomer(int $customerId): array
    {
        return $this->createQueryBuilder('o')
            ->where('o.customer = :customer')->setParameter('customer', $customerId)
            ->andWhere('o.status = :status')->setParameter('status', OrderStatus::Open)
            ->getQuery()->getResult();
    }
}
```

Return domain-meaningful results with named methods; do not leak `QueryBuilder`
fragments to callers. Inject the repository; services depend on *it*, not on the
`EntityManager` directly, where practical.

## The EntityManager and UnitOfWork

- `persist()` is for **new** entities only — it does not "save". Managed entities
  are tracked automatically; you do not persist them again to update.
- `flush()` computes changes for **everything the manager tracks** and writes them
  in one transaction. Call it **once per use case**, at the end — never inside a
  loop (that is a transaction and a round-trip per iteration).
- Wrap multi-step writes in a transaction: `$em->wrapInTransaction(fn () => ...)`.
- Know **managed vs. detached**: an entity from a closed/cleared manager is
  detached; mutating it does nothing until re-managed.

## Querying and hydration

- Use **DQL/QueryBuilder** for portable queries; **always parameterize** — never
  concatenate input into DQL/SQL (`security`). Drop to native SQL only for engine-
  specific needs, still parameterized.
- Match hydration to purpose: hydrate **objects** when you will mutate them;
  hydrate **read models** for display. A DTO query avoids loading entities you only
  read:

```php
// Read model: no managed entities, far cheaper for list/report views
$rows = $em->createQuery(
    'SELECT NEW App\Report\OrderRow(o.id, o.reference, c.name)
     FROM App\Entity\Order o JOIN o.customer c'
)->getResult();
```

Array hydration (`Query::HYDRATE_ARRAY`) is another cheap read path.

## Batch processing and long-running workers

The identity map grows with every entity the manager touches — a memory leak in
loops and workers:

```php
foreach ($iterable as $i => $entity) {
    $this->process($entity);
    if ($i % 100 === 0) {
        $em->flush();
        $em->clear(); // release the identity map
    }
}
```

In Messenger workers, `clear()` (or reset the manager) between messages, and be
aware that a long-lived manager can serve **stale** entities — refetch when
freshness matters.

## Migrations

- Generate migrations from the mapping diff (`make:migration`) but **review the
  SQL** — never run generated migrations blindly; they can produce destructive or
  locking DDL.
- **Never edit a shipped migration** — add a new one.
- Apply the expand-contract discipline for live tables and prove up *and* down on a
  copy (`database-design`, `verify`).

## Calibration — worked examples

| Read pattern | Fetch / hydration |
|---|---|
| Load rows you will mutate | Object hydration, lazy associations. |
| Use a related entity per row | Fetch-join it (avoid N+1). |
| List/report view, read-only | DTO (`SELECT NEW`) or array hydration. |
| Count or paginate a big collection | `EXTRA_LAZY`. |
| Process thousands in a job | Batch + `flush()`/`clear()` every N. |

## Anti-patterns

- **N+1** — lazy associations loaded in a loop; the default Doctrine perf bug.
- **EAGER by default** — loading graphs nobody asked for.
- **Flush in a loop** — a transaction per iteration.
- **God repository / queries everywhere** — DQL scattered through controllers and
  services instead of named repository methods.
- **Unbounded identity map** — no `clear()` in batch jobs and workers.
- **Blindly running generated migrations** — unreviewed destructive/locking DDL.
- **Entities as API DTOs** — serializing managed entities across the boundary
  instead of a read model (`api-design`).

## Where this fits

`doctrine` is the Layer 3 persistence skill under `symfony`. It realizes the schema
from `database-design`, is a primary subject of `performance` (N+1, hydration,
batching), depends on `security` for query safety, and is exercised through the
repository boundary that `architecture` and `engineering-standards` prescribe.
