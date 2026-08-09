---
name: database-design
description: >-
  Relational schema design and data modeling — model for correctness first,
  enforce integrity in the database, index deliberately, and migrate safely.
  Use when designing a schema, choosing keys and constraints, modeling
  relationships, planning indexes, writing migrations, handling temporal/audit
  data, or deciding when to denormalize. ORM-agnostic, with expand-contract
  migration safety; see your stack plugin for ORM-specific mapping notes.
  Triggers: "database schema", "data model", "normalize", "add a table/column",
  "index this", "migration", "foreign key", "should I denormalize", "soft
  delete".
---

# Database design

The database usually outlives the application code on top of it, so its structure
is one of the most expensive things to get wrong (`architecture`, one-way door).
Design for **correctness first** — a normalized, constrained schema that cannot
hold invalid data — and denormalize only under measured pressure, never
speculatively.

## Model for correctness first

- Normalize to third normal form as the default: every fact in one place, no
  update anomalies. Duplication in a schema is a bug waiting to diverge.
- **Denormalize deliberately** — only when profiling shows a real read cost, and
  then document why and how the copy stays consistent. Denormalization is a
  performance trade (`performance`), not a starting point.

## Keys and constraints

- **Surrogate primary key** by default (auto-increment `BIGINT`, or UUIDv7 when you
  need client-generated or distributed-friendly ordered ids). Keep natural keys as
  `UNIQUE` constraints, not as the primary key.
- **Push integrity into the database** — `NOT NULL`, `UNIQUE`, `FOREIGN KEY`,
  `CHECK`. Application validation is the first line; the database is the last line
  and the only one that holds under concurrency and bad actors. Never rely on app
  code alone for integrity.
- Foreign keys with explicit `ON DELETE` behavior (restrict by default; cascade
  only where the child truly cannot exist without the parent).

## Relationships and indexing

- Model relationships by cardinality; use a join table for many-to-many, never
  comma-separated ids in a column.
- **Index what you query** — columns in `WHERE`, `JOIN`, and `ORDER BY`. In a
  composite index, order columns most-selective/most-used first, and remember it
  serves left-prefix queries only.
- **Do not over-index** — every index is write and storage cost. Add indexes to
  match real query patterns, and drop ones the query planner never uses.

```sql
-- Declare the indexes the schema needs, near the table/entity definition
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
ALTER TABLE orders ADD CONSTRAINT uniq_orders_reference UNIQUE (reference);
```

Most ORMs let you declare indexes/constraints directly on the entity/model
mapping instead of a separate DDL file — see your stack plugin for the syntax.

## Migration safety

A migration runs against real data, often on a large live table — treat it like
the irreversible operation it is (`verify` proves it up *and* down on a copy).

- **Expand → migrate → contract** for zero-downtime changes: add the new
  column/table (nullable, additive), backfill in batches separately, switch the
  code, then remove the old — never rename-in-place under load.
- Avoid operations that lock large tables during peak (adding a non-null column
  with a default on old MySQL, blocking index builds). Know your engine's locking.
- Every migration must be reversible, or its `down()` must be a deliberate,
  documented decision.

## Naming and temporal data

- One consistent convention: `snake_case`, and pick singular *or* plural table
  names and hold to it. Foreign keys named predictably (`customer_id`).
- Standard timestamps: `created_at`, `updated_at` (immutable audit of row life).
- **Soft deletes** (`deleted_at`) only when you truly need recoverability or
  history — they leak into every query and are easy to forget. Prefer a real audit
  log or archive table when the requirement is "keep history."

## Calibration — worked examples

| Decision | Default | When to deviate |
|---|---|---|
| Normalize or denormalize | Normalize (3NF) | Denormalize only on measured read pressure, documented. |
| Primary key | Surrogate (BIGINT / UUIDv7) | Natural key as `UNIQUE`, rarely as PK. |
| Enforce a rule | DB constraint + app validation | — always both; never app-only. |
| Add an index | Match a real query | Drop if the planner ignores it; avoid on hot-write, low-read columns. |
| Delete rows | Hard delete | Soft delete only for genuine recoverability/history needs. |
| Schema change on a live table | Expand-contract, batched backfill | — never in-place rename/lock under load. |

## Anti-patterns

- **App-only integrity** — no DB constraints, trusting code to keep data valid.
- **EAV / god tables** — modelling everything as key-value to avoid schema changes.
- **String-list columns** — comma-separated ids instead of a relation.
- **Index-everything or index-nothing** — both are guesses; index to real queries.
- **In-place migration under load** — renames and locking DDL on big live tables.
- **Reflexive soft delete** — `deleted_at` everywhere, quietly corrupting every
  query that forgets it.

## Where this fits

`database-design` is a Layer 2 skill feeding the data decisions in
`feature-delivery` and the structure set by `architecture`. Migration proof lives
in `verify`, query-cost tuning in `performance`, and the entity-level
conventions in `engineering-standards` and your stack plugin's ORM skill.
