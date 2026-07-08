---
name: composer
description: >-
  Composer dependency management — changelog-first updates, semver constraints,
  lock-file hygiene, security auditing, and replacing abandoned packages. Use
  when updating dependencies, auditing for vulnerabilities, managing
  composer.lock, resolving version conflicts, configuring automated updates
  (Dependabot/Renovate), or adding/replacing a package. Enforces the rule that
  dependencies are never changed without approval. Triggers: "composer update",
  "add a package", "composer audit", "version conflict", "composer.lock",
  "abandoned package", "Dependabot", "dependency update".
---

# Composer

Dependencies are code you did not write but are fully responsible for. Manage them
deliberately: **update changelog-first, keep the lock file honest, audit for
vulnerabilities, and never add, remove, or upgrade a dependency without explicit
approval** (`engineering-standards`). When proposing one, justify the need and list
alternatives; prefer well-maintained, widely-adopted, ideally official packages.

## The changelog-first update workflow

Never run a blind `composer update` (it moves everything at once, unreviewed).
Instead:

1. `composer outdated --direct` — see what your *direct* dependencies could move to.
2. Read the changelog / release notes for each candidate before updating it.
3. Update deliberately — one package (with its dependencies) at a time for anything
   non-trivial: `composer update vendor/pkg --with-dependencies`.
4. Run the suite and boot the app after each meaningful update (`verify`); batch
   only low-risk patch/minor bumps together.

## Semver constraints

Be intentional about what you allow:

- `^1.2` — up to (not including) `2.0`; the sensible default for libraries following
  semver.
- `~1.2.3` — up to (not including) `1.3.0`; tighter, patch-level only.
- Pin exact versions only for known-fragile packages; over-pinning blocks security
  patches. Loosen deliberately, not by habit.

## Lock-file hygiene

- **Commit `composer.lock`** for applications — it is the source of truth for
  reproducible installs.
- CI and production run `composer install` (installs exactly the lock); only
  `composer update` changes the lock, and only locally, deliberately.
- **Never hand-edit `composer.lock`** (`engineering-standards`). Review the lock
  diff in every PR that changes dependencies — an unexpected transitive bump is a
  signal, not noise.

## Security

- Run `composer audit` in CI and locally; it flags known CVEs in your tree.
  Vulnerable dependencies are an OWASP Top 10 category on their own (`security`).
- Patch known-vulnerable packages promptly — a security patch jumps the normal
  approval queue, but still gets reviewed and verified.

## Abandoned packages and platform

- Composer warns when a package is **abandoned**; treat it as debt — plan a
  migration to the suggested replacement rather than pinning forever on unmaintained
  code.
- Pin the runtime with `config.platform.php` so CI resolves the same versions your
  production PHP allows (coordinates with `php-upgrade`).

## Automated updates

- Configure **Dependabot or Renovate** to raise PRs: group patch/minor updates,
  keep majors as individual PRs for manual review. Automation surfaces updates; a
  human still reads changelogs and approves majors.
- Automated PRs must pass the full suite before merge — the bot proposes, `verify`
  disposes.

## Calibration — worked examples

| Update type | Process |
|---|---|
| Patch (bugfix) | Batch, run suite, review lock diff. |
| Minor (new features, BC) | Read notes, update, run suite + app. |
| Major (breaking) | One at a time; read UPGRADE; approval required. |
| Security patch | Prompt, still reviewed and verified. |
| New dependency | Approval + justification + alternatives first. |
| Abandoned package | Plan replacement; do not pin indefinitely. |

## Anti-patterns

- **Blind `composer update`** — moving the whole tree unreviewed.
- **Hand-editing the lock file** — corrupting the source of truth.
- **Merging without reading the lock diff** — missing a silent transitive bump.
- **Adding dependencies without approval** — or without weighing alternatives.
- **Over-pinning** — exact versions everywhere, blocking security patches.
- **Ignoring `composer audit`** — shipping known CVEs.
- **Dev dependencies in production** — install with `--no-dev` for prod builds.

## Where this fits

`composer` is the Layer 3 dependency workflow enforcing the dependency discipline
of `engineering-standards`. It gates on `verify` after every update, feeds
`security` (audit, CVE patching), and coordinates with `php-upgrade`
(`config.platform`) and `symfony-upgrade` (bundle compatibility) during version
moves.
