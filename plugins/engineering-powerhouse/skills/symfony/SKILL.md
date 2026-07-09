---
name: symfony
description: >-
  The idiomatic way to build with Symfony 7.x on PHP 8.3+ — where your code
  belongs, which component to reach for, the dependency-injection conventions,
  and the pitfalls that bite. Use when implementing or troubleshooting anything
  Symfony: controllers, routing, forms, validation, services/DI, events,
  Messenger, Console, Twig, Serializer, Security, HTTP client, or cache. Focuses
  on house conventions and pitfalls, not re-documenting symfony.com. Triggers:
  "Symfony", "controller", "routing", "form", "validator", "service", "autowire",
  "event listener", "Messenger", "console command", "Twig", "serializer".
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
