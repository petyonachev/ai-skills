---
name: code-review
description: >-
  Interactive, level-calibrated, multi-agent review of the current code changes.
  Invoked as a command (/code-review). Asks what depth of review is needed
  (hotfix, quick, medium, extensive), then fans out review agents across the
  relevant aspects — stability, security, cleanliness, tests, performance — using
  the standards and design skills as criteria, and reports each issue with its
  location and a short fix. Use to review a diff before committing or opening a
  PR. Triggers: "/code-review", "review my changes", "review this code", "review
  before PR", "check the diff for issues", "review this branch".
---

# Code review

An interactive review of the current changes that reports **actionable issues,
calibrated to how much polish the change actually needs**. A hotfix and a
flagship feature do not deserve the same review, and a review that buries a
real bug under fifty style nits is worse than useless. This workflow scales its
strictness to the situation and fans out across review aspects so nothing
important is missed.

It runs on `parallel-agents` (the aspect fan-out) and uses the standards, design,
security, and testing skills as its criteria — it does not re-derive what "good"
means, it applies those skills at the chosen strictness.

## 1. Scope the change

Determine exactly what is under review before anything else:

- Default to the working diff plus staged changes; for a branch, the diff against
  the base (`git diff main...HEAD`).
- Confirm the scope with the user if ambiguous. **Review only the change** — do not
  audit the whole codebase or flag pre-existing issues outside the diff (note them
  separately at most).

## 2. Ask the review level

Ask the user which review is needed (use an interactive question when the session
is interactive). The four levels and what each optimizes for:

- **Hotfix** — must ship fast and cannot break. Enforces stability and safety only;
  ignores patterns and style entirely. "Is this correct and safe to deploy right
  now?"
- **Quick** — make sure it works well, even if not perfectly clean. Correctness,
  security, and clear defects; light on cleanliness, skips minor smells.
- **Medium** — balances delivery and quality. Enforces good patterns, catches
  meaningful code smells, and filters problematic code, but does not chase the
  tiniest nits. The sensible default for normal PRs.
- **Extensive** — ship very clean code. Enforces all good patterns and reports even
  minor smells and nitpicks; nothing gets a pass.

## 3. Map level to aspects and strictness

Each level activates a set of review aspects and a **noise floor** — the minimum
severity worth reporting. Below the floor, stay silent.

| Level | Aspects reviewed | Report at or above | Skips |
|---|---|---|---|
| **Hotfix** | Stability/correctness, security (critical) | High | All style, patterns, smells, tests-nice-to-have. |
| **Quick** | Stability/correctness, security | Medium (Critical/High any aspect) | Minor smells, naming, style. |
| **Medium** | Stability, security, cleanliness & design, tests | Medium | Trivial nits, subjective style. |
| **Extensive** | All of the above + performance + design/architecture | Low / Nit | Nothing — report everything. |

Aspect → criteria skill the reviewing agent applies:

- **Stability / correctness** → `debug`, `verify`, `testing` lens — logic errors,
  edge cases, error handling, regressions, breaking changes, race conditions.
- **Security** → `security` — injection, authz/IDOR, secrets, input validation.
- **Cleanliness / design** → `code-quality`, `solid`, `refactoring-catalog`,
  `design-patterns`, `engineering-standards` — smells, naming, SOLID, patterns,
  readability, layering.
- **Tests** → `testing` — is new logic covered, are the tests meaningful, edge
  cases missed.
- **Performance** → `performance` — N+1, query cost, caching, hot paths.
- **Design / architecture** → `architecture` — boundary and dependency violations.

## 4. Dispatch review agents

Fan out one agent per active aspect (`parallel-agents`) so each reviews the whole
diff through a single, deep lens. For a tiny diff a single pass may cover it — do
not over-spawn (`parallel-agents` calibration). Brief each agent with:

- The exact scope (the diff), the chosen **level and its noise floor**, the aspect
  it owns and the skill(s) to apply, and the finding format below.
- The instruction to report only issues **within the diff** and **at or above the
  noise floor** — a quiet aspect returning "no issues" is a valid, useful result.

## 5. Aggregate and verify

Do not dump raw agent output:

- **Dedupe** issues multiple agents flagged; **merge** overlapping ones.
- **Cut false positives** — for every Critical/High finding, confirm it is real
  against the actual code before reporting (an adversarial check via
  `parallel-agents`/`verify`). A confident-but-wrong finding erodes trust in the
  whole review.
- **Sort by severity**, most severe first.

## 6. Report

Lead with a summary, then the issues, then a verdict:

```
## Code review — <level> · <scope>
Critical: N · High: N · Medium: N · Low: N

### Critical
[CRITICAL] <one-line title> — stability
  Location: src/Order/OrderService.php:142
  Issue: <one sentence: what is wrong and why it matters>
  Fix: <short, concrete how-to>

### High
[HIGH] <title> — security
  Location: ...
  Issue: ...
  Fix: ...

...

### Verdict
<Ship / Fix-first / Blocked> — <one-line rationale>
```

Every issue names **what**, **where** (`file:line`), and **how to fix** in a
sentence. Keep fixes concrete enough to act on without a follow-up question. If an
aspect found nothing, say so briefly rather than omitting it silently.

## Severity model

- **Critical** — breaks production or exposes data; must fix before ship.
- **High** — significant bug, security gap, or major design flaw; fix before ship.
- **Medium** — real smell, weak/missing test, moderate issue; fix per level.
- **Low / Nit** — minor style, naming, subjective polish; reported only at
  Extensive.

## Anti-patterns

- **Noise over signal** — burying a real bug under style nits; the noise floor
  exists to prevent this.
- **Wrong strictness** — style-nitpicking a hotfix, or waving through a flagship
  feature.
- **Scope creep** — reviewing code outside the diff, flagging pre-existing issues
  as if introduced.
- **Unverified findings** — reporting plausible-but-wrong Critical/High issues.
- **Vague findings** — "improve error handling" with no location or concrete fix.
- **Over-spawning** — a fleet of agents for a three-line diff.

## Where this fits

`code-review` is a Layer 1 workflow. It orchestrates `parallel-agents` and applies
the criteria skills (`code-quality`, `solid`, `refactoring-catalog`,
`design-patterns`, `security`, `testing`, `performance`, `architecture`,
`engineering-standards`), verifying findings through `verify`. It complements
`finish-branch` (run it before shipping) and `feature-delivery`'s self-review step
(this is the deeper, external pass).
