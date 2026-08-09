---
name: performance
description: >-
  Profiling-driven performance optimization — measure first, fix the real
  bottleneck (usually I/O, not CPU), then verify the win. Use when the app is
  slow, latency is high, queries are slow, you hit an N+1, need a caching
  strategy, must tune database access, or want to set performance budgets.
  Language/ORM-agnostic; see your stack plugin for tool-specific profiling and
  ORM notes. Triggers: "it's slow", "optimize", "reduce latency", "slow query",
  "N+1", "caching", "performance tuning", "profiling", "load test", "memory
  usage".
---

# Performance

Performance work is governed by one rule that overrides intuition: **measure
first, optimize the bottleneck the measurement reveals, then measure again to
confirm the win.** Optimizing by guessing wastes effort on code that was never
slow and often makes things worse. Equally, premature optimization is a defect —
it trades the clarity you always need for speed you may never need
(`engineering-standards`).

## The profiling loop

Run it as an `iterate` loop with a measured signal, never a hopeful edit:

```
measure (profile / query count / timing) → find the real hot path →
change one thing → measure again → keep only what helped
```

Use real tools — your framework's profiler or an APM tool for request timing and
query counts, `EXPLAIN` for query plans, a memory profiler for leaks. A change
with no before/after number is not an optimization, it is a guess (`debug`,
performance category).

## Where the time actually goes

In a typical web app the wins are overwhelmingly **I/O, not CPU**:

- Database access (N+1, missing indexes, over-fetching) — usually the biggest.
- External/API calls (`integration-build`) — network latency, no caching.
- Serialization and rendering of oversized payloads.

Micro-optimizing application-code loops while a request fires 400 queries is
effort in the wrong place. Follow the measurement to the dominant cost.

## The N+1 problem

The single most common ORM performance bug: iterating a collection and lazily
loading a relation per element — 1 query becomes N+1.

```
// Before — 1 query for orders, then 1 per order for its customer (N+1)
orders = repository.findAll()
for order in orders:
    print(order.customer.name)   // triggers a lazy-load query per order

// After — a single join/eager-load fetches customers with the orders
orders = repository.query()
    .join("customer")
    .fetch()
```

Detect it by watching the profiler's query count; fix with eager-loading/joins,
batch loading, or a lazy-collection mode where you only count/paginate. Your
ORM's specific syntax for this lives in your stack plugin.

## Query optimization

- `EXPLAIN` slow queries; add the index the plan wants (`database-design`).
- Select only the columns/DTO you need — avoid hydrating whole entity graphs for a
  read-only list (array/DTO projection is far cheaper).
- Paginate; never load an unbounded result set into memory.
- For large batch jobs, process in chunks and clear the ORM's identity
  map/session periodically so it does not exhaust memory.

## Caching strategy

Cache from the outside in, and remember the hard part is invalidation:

- **HTTP / reverse proxy / CDN** — cache full responses with `Cache-Control`/`ETag`
  (a reverse-proxy cache, a CDN). Cheapest hit; serves before your code runs.
- **Application cache** — memoize expensive computations and query results
  (Redis or your language's cache abstraction). Use clear, versioned keys; set
  TTLs; invalidate on write.
- Cache the *expensive and stable*; do not cache cheap or rapidly-changing data —
  a stale or thrashing cache is worse than none. Every cache entry needs an answer
  to "how does this get invalidated?"

## Async and budgets

- **Offload slow, non-interactive work** (emails, exports, third-party calls) to
  a background job queue so the request returns fast; the user does not wait on
  what they do not need synchronously.
- **Set budgets/SLOs** (p95 latency, query-count ceilings) and **load-test** before
  scaling — know the target and the actual before adding infrastructure
  (`architecture`, YAGNI).

## Calibration — worked examples

| Symptom | Likely cause | First move |
|---|---|---|
| Endpoint slow, many queries | N+1 | Profiler query count → fetch-join. |
| One query slow | Missing index / bad plan | `EXPLAIN` → index (`database-design`). |
| Slow, few queries, high CPU | Hydration / serialization | Array/DTO hydration; trim payload. |
| Memory blows up in a batch | Whole set in memory | Chunk + `clear()`; paginate. |
| Slow due to third party | Sync external call | Cache the response; offload to Messenger. |
| Repeated identical work | No caching | App cache with a real invalidation plan. |

The pattern: **measure to find the dominant cost, and in a typical web app that
cost is almost always the database or an external call — fix that before
touching application code.**

## Anti-patterns

- **Optimizing without measuring** — guessing at the bottleneck.
- **Premature optimization** — sacrificing clarity for speed nothing needed.
- **Ignoring N+1** — the default ORM perf bug, invisible without the profiler.
- **Cache-everything** — caching volatile data with no invalidation story.
- **Micro-optimizing CPU** while I/O dominates.
- **Scaling before profiling** — throwing hardware at a fixable query.

## Where this fits

`performance` runs on the `iterate` measure-change-measure loop and shares its
root-cause discipline with `debug`. It tunes the schema and indexes from
`database-design`, the payloads from `api-design`, and the external calls from
`integration-build`, all within the budgets and infrastructure restraint of
`architecture`. Optimize only what measurement and `engineering-standards`
(no premature optimization) justify.
