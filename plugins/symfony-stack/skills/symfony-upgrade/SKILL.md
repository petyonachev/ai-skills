---
name: symfony-upgrade
description: >-
  Upgrading Symfony using the deprecation-first approach — drive deprecations to
  zero on the current version, then bump the major. Use when upgrading Symfony to
  a new minor or major, fixing deprecation warnings, updating recipes, checking
  bundle compatibility, or planning a Symfony migration. Covers deprecation
  tracking, the recipe update flow, bundle compatibility, and version-specific
  breaking changes. Triggers: "upgrade Symfony", "Symfony deprecations", "bump
  Symfony version", "update recipes", "migrate to Symfony 7", "LTS upgrade".
---

# Symfony upgrade

Symfony's contract makes upgrades predictable: **a major release removes only what
earlier minors already deprecated.** So the whole strategy is: get current on the
latest minor of your major, drive deprecations to zero there (while everything
still runs), *then* bump the major — which becomes almost a non-event. Skipping
that discipline is what turns upgrades into multi-week ordeals.

Prerequisite: a **green test suite** before you start (`testing`, `verify`). An
upgrade without a safety net is a gamble.

## Minor vs. major

- **Minor** (7.1 → 7.2) — backward compatible. Update the constraint, install, run
  the suite, skim the CHANGELOG for new deprecations. Low risk; do these routinely.
- **Major** (6.4 → 7.0) — removes deprecated code. Risk lives entirely in the
  deprecations you did *not* fix on the last minor. Read `UPGRADE-7.0.md`.

## The workflow

1. **Get current on the latest minor** of your existing major first. Never jump
   two majors at once.
2. **Surface deprecations** — run the test suite with the PHPUnit Bridge and the
   deprecation helper; read the deprecation log in the profiler for runtime paths
   tests miss.
3. **Fix deprecations to zero** — as an `iterate` loop: fix a batch, run the suite,
   repeat until the deprecation count is 0. This is the bulk of the work and it is
   done on the *old* version, where the app still boots.
4. **Bump the major** — raise the `symfony/*` constraints, `composer update`, read
   `UPGRADE-<major>.md` for the handful of removals that had no deprecation path.
5. **Update recipes** — `composer recipes` then `composer recipes:update`; diff the
   config changes and merge deliberately (they are not auto-correct).
6. **Verify** — full suite green *and* boot the app and exercise key flows; config
   and DI changes often pass tests but break at runtime (`verify`).

## Deprecation tracking

- `SYMFONY_DEPRECATIONS_HELPER` controls the PHPUnit Bridge's strictness; use it to
  fail the build on new deprecations once you reach zero, so they cannot creep back.
- The profiler's deprecation panel catches runtime deprecations your tests do not
  exercise — check real request flows, not just the suite.

## Bundle and recipe compatibility

- Before bumping, confirm every third-party bundle supports the target version
  (`composer why-not symfony/framework-bundle <version>` surfaces blockers). A
  single unsupported bundle can gate the whole upgrade — plan its replacement or
  update first.
- Recipes carry framework config; `recipes:update` shows what changed since your
  version scaffolded — review, do not blindly accept.

## Calibration — worked examples

| Move | Effort | Approach |
|---|---|---|
| Patch (7.2.3 → 7.2.7) | Trivial | Update, run suite. |
| Minor (7.1 → 7.2) | Low | Update, run suite, note new deprecations. |
| Major, deprecations already at zero | Moderate | Bump, update recipes, verify. |
| Major, many deprecations outstanding | High | Fix deprecations on the old minor *first*, then bump. |
| Two majors behind (5.4 → 7.x) | High | One major at a time; never skip. |

## Anti-patterns

- **Bumping the major with deprecations unresolved** — inviting the exact breakages
  the deprecation system warned you about.
- **Skipping versions** — jumping two majors instead of stepping through.
- **Upgrading without a green suite** — no way to know what broke.
- **Blindly accepting recipe changes** — overwriting your config.
- **Ignoring runtime deprecations** — only trusting the test suite, missing
  request-path deprecations.

## Where this fits

`symfony-upgrade` is a Layer 3 maintenance workflow. It runs the deprecation-fix
loop via `iterate`, ends at the `verify` gate, coordinates package moves with
`composer`, and often pairs with `php-upgrade` when a Symfony major also raises the
PHP floor.
