---
name: api-design
description: >-
  Designing HTTP APIs (REST and GraphQL) that are consistent, predictable, and
  safe to evolve — resource modeling, methods and status codes, a stable error
  contract, pagination, versioning, and auth. Use when designing a new API,
  reviewing an existing one, choosing REST vs GraphQL, planning versioning,
  defining error responses, or implementing pagination. Triggers: "API design",
  "REST endpoint", "GraphQL", "API versioning", "error response format",
  "pagination", "status code", "idempotency", "API contract".
---

# API design

An API is a contract other people build on, so its cardinal virtue is
**consistency** — a predictable API can be learned once and used everywhere; a
clever-but-inconsistent one is relearned at every endpoint. Design the contract
first, make it uniform, and design for evolution from day one, because you cannot
un-ship a contract consumers depend on (`architecture`, one-way door).

## Resource modeling

- Model **resources (nouns), not actions** — `/orders`, `/orders/42/items`, not
  `/getOrders` or `/createOrder`. The HTTP method is the verb.
- Plural collection names, consistent casing, shallow nesting (two levels max —
  deep nesting couples the URL to your data model). Express further relationships
  with links or query parameters, not `/a/1/b/2/c/3`.

## Methods and status codes

Use the method's defined semantics, and honor idempotency:

- `GET` (safe, cacheable), `POST` (create / non-idempotent action), `PUT` (full
  replace, idempotent), `PATCH` (partial update), `DELETE` (idempotent).
- Return the *right* status: `200/201/204`, `400` (malformed), `401`
  (unauthenticated) vs `403` (unauthorized), `404`, `409` (conflict), `422`
  (validation), `429` (rate limit), `5xx` (server). Never return `200` with an
  error body.
- For unsafe operations that clients may retry (`POST` that charges, sends,
  creates), support an **idempotency key** so a retry does not double-act
  (`integration-build`).

## The error contract

Errors are part of the API. Give them one consistent, machine-readable shape
across every endpoint — RFC 9457 problem+json is a good default:

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "The order must contain at least one item.",
  "errors": [{ "field": "items", "message": "must not be empty" }]
}
```

A stable `type`/machine code lets clients branch on errors; the human message
aids debugging. **Never leak internals** — no stack traces, SQL, or class names in
responses.

## Pagination

- **Cursor-based** for large or frequently-changing collections — stable under
  inserts, no deep-offset cost.
- **Offset/limit** only for small, stable datasets where jumping to a page matters.
- Always bound page size (a sane default and a max); return paging metadata /
  next-cursor links.

## Versioning and evolution

- **Additive changes need no version** — new optional fields, new endpoints.
  Design clients to ignore unknown fields so you can extend freely.
- **Version only on a breaking change** — removing/renaming a field, changing a
  type or semantics. Pick one scheme (URI `/v2/` or a header) and keep it uniform.
- Publish a **deprecation policy**: mark deprecated, announce, support the old
  version for a stated window, then remove. Never break consumers silently.

## REST or GraphQL

- **REST** — resource-oriented, cache-friendly, simplest for CRUD and public APIs.
  The default.
- **GraphQL** — when clients need to shape wildly varying payloads and
  over/under-fetching is a real problem (rich frontends, aggregation). It moves
  complexity server-side (query cost, N+1, auth per field) — adopt it for that
  specific need, not as a default.

## Symfony notes

API Platform gives you REST/GraphQL, OpenAPI docs, pagination, and validation from
your resources — prefer it over hand-rolling. Use Serializer groups to control
representations, the Validator for `422` contracts, and stateless token auth
(OAuth2/JWT) rather than sessions for APIs.

## Calibration — worked examples

| Decision | Default | Deviate when |
|---|---|---|
| REST vs GraphQL | REST | Clients need flexible, varying field selection. |
| Version the API | Not yet (additive) | A breaking change is unavoidable. |
| Pagination style | Cursor | Small stable set where page-jumping matters → offset. |
| Error body | problem+json, uniform | — keep it uniform everywhere. |
| Retryable unsafe op | Idempotency key | — always, for charge/send/create. |
| Auth | Stateless token + scopes | — never roll your own crypto/session for APIs. |

## Anti-patterns

- **Verbs in URLs** — `/createOrder`; the method is the verb.
- **200-with-error-body** — hiding failures behind a success status.
- **Inconsistent errors** — a different error shape per endpoint.
- **Leaking internals** — stack traces / SQL in responses.
- **Breaking without versioning** — silently changing a contract consumers depend
  on.
- **Unbounded pagination** — no max page size, deep-offset scans.
- **GraphQL by default** — adopting it without the over-fetching problem it solves.

## Where this fits

`api-design` is a Layer 2 skill for the contract decisions in `feature-delivery`
and `integration-build` (idempotency, error mapping), sitting under the structure
from `architecture`. Auth depth is in `security`, response performance and N+1 in
`performance`, and framework specifics in the `symfony` stack skill.
