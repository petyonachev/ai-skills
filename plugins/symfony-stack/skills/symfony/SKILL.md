---
name: symfony
description: >-
  The idiomatic way to build with Symfony 7.x on PHP 8.3+ — where your code
  belongs, which component to reach for, the dependency-injection conventions,
  PHP/PSR-12 conventions, PHPUnit testing specifics, Symfony security (Voters,
  CSRF, PasswordHasher), API Platform, and the pitfalls that bite. Use when
  implementing or troubleshooting anything Symfony: controllers, routing, forms,
  validation, services/DI, events, Messenger, Console, Twig, Serializer,
  Security, HTTP client, cache, or testing. This is the language/framework
  specialization of the stack-agnostic core skills. Triggers: "Symfony", "PHP",
  "controller", "routing", "form", "validator", "service", "autowire", "event
  listener", "Messenger", "console command", "Twig", "serializer", "PHPUnit",
  "Voter", "CSRF".
---

# Symfony

Symfony is a set of decoupled components under a framework. You already know the
APIs; this skill is about the *idiomatic* way to use them here, where your code
belongs, and the mistakes that recur — not a restatement of the docs. Target
**Symfony 7.x on PHP 8.3+**, with **attributes as the default** configuration
style.

## Where your code belongs

The request flows front controller → `HttpKernel` → events → controller →
`Response`. Your code slots in with a strict division of labor
(`engineering-standards`):

- **Controllers stay thin** — receive the request, invoke one application service,
  return a response. No business logic, no queries, no orchestration.
- **Business logic lives in the application/service layer** — plain, injectable
  services holding the use case.
- **Entities hold domain state and invariants** — not persistence orchestration,
  not formatting.
- **Validate input at the boundary** — a DTO + Validator, or a Form, before
  anything reaches a service.

## Dependency injection — the heart of the framework

- **Constructor injection, always.** Autowiring + autoconfiguration are on; type-
  hint the dependency and let the container wire it. Never inject the container
  itself or use it as a service locator, and never reach for services statically.
- Use `#[Autowire]` for specific values/services (`#[Autowire('%env(...)%')]`,
  `#[Autowire(service: ...)]`), `#[AutowireIterator]`/`#[AutowireLocator]` for
  tagged collections (Strategy — see `design-patterns`).
- Keep services stateless and `final`. Configuration that varies by environment
  comes from **env vars**; secrets from the **secrets vault**, never from code.

## Which component for which job

| Need | Reach for | Idiomatic note |
|---|---|---|
| Request/response | HttpFoundation | Return typed `Response`; don't touch PHP superglobals. |
| Routing | `#[Route]` attributes on controllers | Name routes; keep controllers thin. |
| Input → object + validation | DTO + `#[MapRequestPayload]`, or Form + Validator | Validate at the boundary; constraints as attributes. |
| Domain reactions | EventDispatcher + `#[AsEventListener]` | Decouple side effects; not for control flow. |
| Slow / background work | Messenger | Offload emails, exports, third-party calls off the request. |
| CLI | Console + `#[AsCommand]` | One command = one use case; delegate to a service. |
| Templating | Twig (autoescape ON) | Every `|raw` is a reviewed decision (`security`). |
| Serialization / API | Serializer (+ API Platform) | Control shape with groups; see `api-design`. |
| AuthN/AuthZ | Security component | Authenticators, Voters, `#[IsGranted]` (`security`). |
| Outbound HTTP | HttpClient | Wrap in an anti-corruption layer (`integration-build`). |
| Caching | Cache (PSR-6/16) | Real invalidation plan required (`performance`). |

## Configuration conventions

- **Attributes over YAML/XML** for routes, listeners, commands, validation,
  mapping — unless the module you are in already standardizes on config files
  (`architecture`: match the local style).
- `config/services.yaml` for wiring that cannot be autowired; keep it minimal.
- Env vars for deploy-varying values; typed env var processors (`%env(int:...)%`)
  where useful; `.env` for defaults only, never for secrets.

## PHP and Symfony conventions

- `declare(strict_types=1)` in every PHP file. Follow PSR-12.
- Use modern PHP (8.3+): `readonly` properties, enums, named arguments,
  first-class callable syntax where they improve clarity.
- **Constructor injection** for dependencies — no service location, no static
  access to services (see the DI section above).
- **Ticket-prefixed branches/commits are the norm here**: `PROJ-123-name-of-the-
  branch`, commit subject `PROJ-123 name of the commit` — see
  `engineering-standards` for the general form this specializes.

## Testing (PHPUnit)

- Use **attributes** (PHPUnit 10+/11): `#[Test]`, `#[DataProvider('provider')]`
  (the provider is a `static` method), `#[CoversClass(...)]`, `#[Group(...)]`.
- **Data providers** for the same behavior across many inputs — one test method,
  many cases, each named.
- **Doubles**: `createStub()` for query-only collaborators, `createMock()` when you
  must assert interactions; prefer stubs unless the interaction *is* the behavior.
- **Symfony test cases**: `KernelTestCase` when you need the container/services;
  `WebTestCase` for HTTP functional tests; `CommandTester` for console commands.
- **Database tests**: run each inside a transaction rolled back in `tearDown`, or
  use automatic per-test rollback, so tests never see each other's data. Test
  against a real (test) database, not a mocked repository, when the query itself
  is what you are verifying.
- See the stack-agnostic `testing` skill for the strategy (pyramid, doubles,
  isolation) this specializes.

## Security

- **Input/injection**: parameterize DQL/SQL — never concatenate. The Doctrine
  QueryBuilder does this for you: `->where('u.email = :email')->setParameter('email', $email)`.
  Raw DQL/SQL, `LIKE` fragments, and dynamic `ORDER BY` are still injectable.
- **Output/XSS**: Twig auto-escapes HTML by default — **keep it on** and treat
  every `|raw` as a reviewed decision, never a convenience.
- **CSRF**: protect state-changing form/browser requests with Symfony's form CSRF
  / `IsCsrfTokenValid`.
- **Auth**: hash passwords with argon2id/bcrypt via Symfony's `PasswordHasher`,
  never roll your own.
- **Authorization**: enforce ownership/permission explicitly with a Voter —

  ```php
  if (!$this->isGranted('ORDER_VIEW', $order)) {
      throw $this->createAccessDeniedException();
  }
  ```
- **Secrets**: load from environment / the Symfony secrets vault, never hardcode.
- **Dependencies**: run `composer audit` in CI (`composer` skill).
- See the stack-agnostic `security` skill for the full OWASP-level threat model
  this specializes.

## API Platform

API Platform gives you REST/GraphQL, OpenAPI docs, pagination, and validation from
your resources — prefer it over hand-rolling. Use Serializer groups to control
representations, the Validator for `422` contracts, and stateless token auth
(OAuth2/JWT) rather than sessions for APIs. See the stack-agnostic `api-design`
skill for the underlying contract decisions.

## Pitfalls that recur

- **Fat controllers** — logic that belongs in a service, done in the controller
  because it was faster to type.
- **Container as service locator** — injecting `ContainerInterface` and pulling
  services out; inject what you need instead.
- **Unvalidated request handling** — reading raw request data into a service with
  no DTO/Form/validation boundary.
- **EAGER associations by default** — silently loading object graphs
  (`database-design`, `performance`).
- **Events for control flow** — using the dispatcher to sequence steps that should
  be an explicit method call; events are for decoupled *reactions*.
- **Sync slow work** — sending mail or calling a third party inside the request
  instead of dispatching to Messenger.
- **Sessions for API auth** — APIs should be stateless token-authenticated
  (`api-design`, `security`).

## Deeper reference

For depth on the two areas where the house-idiomatic approach with worked
examples matters most (progressive disclosure — not loaded until needed):

- `references/di-and-config.md` — the container, autowiring, tagged services,
  factories, decoration, parameters, and env vars.
- `references/forms-and-validation.md` — request DTOs and `#[MapRequestPayload]`,
  the Validator, and Forms, with the boundary-validation conventions.

## Anti-patterns

- **Reinventing framework features** — hand-rolling DI, events, or a command bus.
- **Business logic in controllers or entities** — it belongs in services.
- **Static/service-locator access** — breaks testability and hides dependencies.
- **Config sprawl** — mixing attributes and YAML for the same concern within a
  module.
- **Ignoring Messenger** — doing slow work synchronously.

## Where this fits

`symfony` is the Layer 3 stack anchor that the flagship workflows lean on for the
"how" of implementation. It realizes the layering of `engineering-standards`, the
patterns of `design-patterns`/`solid`, and the contracts of `api-design`, and
pairs with `doctrine` for persistence, `composer` for dependencies, and
`symfony-upgrade`/`php-upgrade` for version moves.
