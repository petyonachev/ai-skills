---
name: php-upgrade
description: >-
  Upgrading PHP (8.0 → 8.4+) with a changelog-first, tooling-assisted workflow —
  read the migration guide, work behind a green test suite, automate with Rector,
  audit with PHPCompatibility. Use when upgrading the PHP version, checking
  compatibility, fixing deprecations, running Rector, or planning a PHP migration.
  Covers per-version breaking changes, php.ini/extension compatibility, Rector
  rule sets, and the deployment/runtime bump. Triggers: "upgrade PHP", "PHP
  compatibility", "Rector", "PHP deprecations", "migrate to PHP 8.x", "bump PHP
  version".
---

# PHP upgrade

A PHP upgrade is mechanical and low-risk *if* done changelog-first behind a test
net, and error-prone if done by guessing. The method: know the breaking changes
for the target version, run behind a **green test suite** (`testing`, `verify`),
automate the repetitive refactors with Rector, and verify against the new runtime
before shipping.

## The workflow

1. **Read the migration guide** for the target version — know its breaking changes
   and new deprecations before touching code.
2. **Confirm the safety net** — a green suite. If coverage is thin, characterize the
   risky areas first (`refactor-safely`, `testing`).
3. **Audit compatibility** — run PHPCompatibility (via PHP_CodeSniffer) targeting
   the new version to list what breaks, and Rector in **dry-run** to preview
   automated fixes.
4. **Apply fixes** — run Rector's version rule set, then work the remaining manual
   breakages as an `iterate` loop: fix, run suite, repeat until green. **Review
   Rector's diff** — never blind-commit generated changes.
5. **Update runtime config** — `php.ini` settings and extension versions for the
   target; check every extension supports it.
6. **Verify on the new runtime** — run the suite *on the target PHP version* and
   boot the app; passing on the old version proves nothing about the new one.

## Tooling

- **Rector** — automates the bulk. Apply the version-set (e.g. `LevelSetList::UP_TO_PHP_83`)
  incrementally, one level at a time, tests green after each. It is a power tool,
  not an oracle: read its diffs.
- **PHPCompatibility** — a phpcs standard that flags version-incompatible syntax and
  functions without running the code. Good for the up-front audit and for CI.
- Put both in CI so regressions and new incompatibilities are caught automatically.

## Per-version highlights

Know what the target introduces and removes (read the guide for the full list):

- **8.1** — enums, `readonly` properties, `never`, fibers, first-class callables.
- **8.2** — `readonly` classes, DNF types; dynamic properties **deprecated** (add
  `#[AllowDynamicProperties]` or fix); `${}` string interpolation deprecated.
- **8.3** — typed class constants, `json_validate()`, `#[Override]`, granular
  `Date`/`Random` changes.
- **8.4** — property hooks, asymmetric visibility, `new Foo()->bar()` without
  parens; **implicitly-nullable parameter types deprecated** (`?T` must be explicit).

## Calibration — worked examples

| Move | Effort | Approach |
|---|---|---|
| Patch (8.3.4 → 8.3.11) | Trivial | Update runtime, run suite. |
| One minor (8.2 → 8.3) | Low–moderate | Rector level + guide review + verify on target. |
| Two minors (8.1 → 8.3) | Moderate | Step through levels; do not jump. |
| Legacy jump (7.4 → 8.3) | High | Sizeable breaking changes; Rector + manual, incremental, heavy verification. |

## Anti-patterns

- **Upgrading without tests** — no way to know what the new version broke.
- **Blind Rector commits** — accepting generated diffs unread.
- **Skipping the changelog** — discovering breaking changes in production.
- **Ignoring deprecations** — this version's deprecation is next version's fatal.
- **Verifying on the old runtime** — bumping prod PHP before proving the code on it.
- **Upgrading the runtime image before the code** — flip the order and it breaks.

## Where this fits

`php-upgrade` is a Layer 3 maintenance workflow. It runs the fix loop through
`iterate`, gates on `verify` (on the *target* runtime), leans on `testing` for the
safety net, and coordinates with `composer` (`config.platform`, dependency PHP
constraints) and `symfony-upgrade` when a framework major raises the PHP floor.
