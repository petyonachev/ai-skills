---
name: code-review-protocol
description: >-
  Internal protocol for the code-review topic agents (review-architecture,
  review-reusability, review-safety, review-scalability, review-simplicity,
  review-security, review-tests): the evidence rules that keep findings free of
  false positives, the severity/class model, the level floors, the
  reviewer-pair debate rounds, and the exact output formats. Preloaded into
  those agents and loaded by the code-review orchestrator to parse their
  output. Not a standalone review workflow — to review changes use
  code-review.
user-invocable: false
---

# Code review protocol

This is the contract every topic reviewer follows and the orchestrator
(`code-review`) relies on. It exists for one reason: **the review is the last
gate before production, so every finding must be true.** A false positive costs
trust and time; a false negative ships a defect. The protocol minimizes both —
two independent reviewers per topic find, then argue every finding against the
actual code until only verified claims remain.

## Your situation (as a topic reviewer)

- You own **one topic** and play **one role**, A or B. Another reviewer owns the
  same topic with the other role. You never message them directly — the
  orchestrator relays each side's output verbatim.
- You run in **rounds**. Each round your final message is the entire result of
  that round; the orchestrator resumes you for the next one with your context
  intact. Keep your evidence in mind — you will be asked to defend it.
- You are **read-only** (see "Commands" below).

## The review packet

The orchestrator gives you a packet directory. Read it before anything else:

- `meta.md` — level, topic list, base and head refs, the exact command that
  reproduces the diff, detected stack and the stack skills available.
- `intent.md` — what the change is meant to do: user statement, commit
  messages, PR description, ticket. Judge the code against this intent.
- `files.txt` — changed files with status (A/M/D/R) and line counts, plus the
  files excluded from review (generated, vendored, lockfiles) and why.
- `diff.patch` — the full diff, the snapshot every reviewer reviews.

Read all of `diff.patch` (in chunks if large). Then open the post-change files
themselves — a hunk without its surrounding function is not enough to judge
anything. If the stack has a matching stack skill (e.g. `symfony-stack:doctrine`,
`python-stack:async-python`) and your topic touches it, load it with the Skill
tool — framework behavior is checked against those skills and the installed
library code, never against memory.

## The prime directive — no false positives

A finding is a **claim you have verified against the code**, not a suspicion.
Every finding must survive these rules:

1. **Read before you claim.** Read the whole enclosing function/class in the
   post-change file, the callers that reach it, and the callees and config it
   depends on. Most false positives come from judging a hunk out of context.
2. **Look for the guard before claiming its absence.** Upstream validation,
   middleware/firewalls, framework defaults (auto-escaping, parameterized ORMs,
   CSRF tokens), type systems and non-null declarations, DB constraints, an
   outer try/catch, a feature flag, a retry wrapper. **Every absence claim
   ("no validation", "no test", "no index", "no auth check", "not reused") must
   cite the search you ran** — the pattern, the scope, and the result.
3. **Verify library and framework behavior from the installed version.** Read
   it in `vendor/`, `site-packages`, `node_modules`, the lockfile version, or a
   loaded stack skill. Do not assert what an API does from memory. If you cannot
   verify it, it is a question, not a finding.
4. **Stay in scope.** The code must be added or modified by the diff, or the
   diff must change its behavior (a changed signature reaching an unchanged
   caller, a removed guard exposing an existing path, a migration altering data
   an unchanged query reads). A problem that pre-dates the change and that the
   change does not touch is not a finding — list it under PRE-EXISTING at most
   (max 3, only if serious).
5. **Concrete, not speculative.** A failure-class finding needs a scenario: the
   concrete input, state, or sequence → the observable wrong outcome, on a path
   that is reachable in this codebase today. "If someone later..." is not a
   finding. A quality-class finding needs a concrete present cost: the existing
   code it duplicates (path:line), the boundary it crosses, the specific change
   it makes harder.
6. **Quote exactly.** The `Code` field is copied from the post-change file at
   the cited lines (re-open the file to copy it; for deleted code, quote the
   removed lines and mark the location `(removed)`). Line numbers are
   post-change line numbers.
7. **Respect stated intent.** A deliberate behavior change described in
   `intent.md` is not a defect by itself. Flag it only if the evidence shows it
   is wrong or unsafe — and say why the intent does not cover it.
8. **Stay in your topic.** Something real but owned by another topic goes in
   HANDOFF as one line (location + what to check), never as your finding. The
   orchestrator routes it. This prevents the same issue being double-counted.
9. **Unsure means QUESTION.** A concern you could not verify goes in QUESTIONS,
   phrased as what the author must confirm. Questions are never findings and
   never count toward the verdict.
10. **Silence is a valid result.** There is no quota. "No findings" from a
    careful review is worth more than one invented finding.

### Self-check before emitting each finding

- [ ] I re-opened the file; the quoted code matches the cited lines exactly.
- [ ] The code is in scope (rule 4).
- [ ] I searched for the guard / existing handling / existing test, and cite it.
- [ ] I traced a real caller path to the code (failure class), or cite the
      concrete present cost (quality class).
- [ ] Severity and class follow the rubric below, not intuition.
- [ ] The fix is concrete, and I checked it would not break another caller.
- [ ] It is my topic and at or above the level floor.

If any box is unchecked: fix it, demote it to a QUESTION, or drop it.

## Class, severity, and level

Every finding has a **class** and a **severity**.

**Class** — what kind of problem it is:

- **failure** — the change can make production fail: a crash or error, a wrong
  result, data loss or corruption, a security breach, an outage or resource
  exhaustion, a failed deploy or migration, broken existing behavior or tests.
- **quality** — the code works but carries a real, present cost: a design or
  boundary problem, duplication of existing code, needless complexity, a
  missing or meaningless test. Blocks the PR at Basic and Extended.
- **nit** — a non-blocking improvement; the code is acceptable as-is.

**Severity** — how much it matters:

| Severity | Meaning | Allowed classes |
|---|---|---|
| **Critical** | Will break production, lose/corrupt data, or expose data / allow unauthorized access, on a realistic path. Must not ship. | failure |
| **High** | Significant defect or risk on a realistic path; or a quality problem whose cost is large and hard to undo (a public contract, a boundary that will spread). Fix before merge. | failure, quality |
| **Medium** | A real, concrete problem with limited impact: a rare-path mishandling, an untested case in otherwise tested code, duplicated existing code, an unnecessary abstraction. Fix before merge. | failure, quality |
| **Nit** | Optional polish. Never blocks. | nit |

Judge severity by **impact × likelihood on a reachable path**, with the
topic-specific anchors in your agent definition.

Severity is a property of the defect, not of the review:

- **Level-independent.** The level decides only *whether* a finding is
  reported, never *how severe* it is. The same defect gets the same severity at
  Hotfix, Basic, and Extended.
- **Not cumulative.** A defect raised by several topics is still one defect;
  its severity is the highest any single topic justifies on its own impact.
  Overlapping topics, agreeing reviewers, or merged evidence never raise it.
- **Demonstrated, not imagined.** When two anchors fit, pick the one matching
  the impact you showed on a reachable path, not the worst case you can
  construct.

**Calibration** — borderline cases that are easy to rate inconsistently. These
take precedence over a topic's own anchors when both apply:

| Case | Severity |
|---|---|
| Wrong error type on a request that fails either way (an unhandled 500 where a 404 belongs), nothing exposed or written | Medium |
| A correct value rendered wrongly in output that leaves the system (export, API response, invoice, email) — e.g. `$10.5` for 1050 cents | High |
| A wrong value *computed* (money, quantity, totals) on a main path | Critical |
| A hazard no current input can trigger (unquoted CSV where no field can contain a comma, a missing guard on a value the schema forbids) | Nit |
| Behavior that contradicts the stated intent but could be deliberate (a default, a filter, a cap) | Medium, plus a question to the author |
| A new endpoint's response shape inconsistent with its siblings, no consumer yet | Medium |
| The same, on a released or versioned contract that clients already use | High |
| A new endpoint or public function with no tests at all | High |
| New logic inside a tested unit, untested branch or case | Medium |
| A test that asserts nothing the change could break (header-only, `assertTrue(True)`) | Medium |
| A disabled or skipped existing test without a recorded reason, still passing | High (Critical if it catches a failure the change ships) |

**Levels** — which topics run and which findings are reported:

| Level | Topics reviewed | Report |
|---|---|---|
| **Hotfix** | Safety, Security, Scalability, Tests | **failure class only** (Critical/High/Medium). No quality, no nits. |
| **Basic** | All seven | failure + quality (Critical/High/Medium). **No nits.** |
| **Extended** | All seven | Everything, including nits. |

Below the floor for the level, stay silent — do not report it, do not mention
it. Your agent definition says what your topic covers at each level.

## Roles — two routes through the same diff

Both reviewers cover the whole diff with the same standard; they take
different routes so that one's blind spot is the other's path:

- **Role A — diff-first.** Walk the diff hunk by hunk in file order. For each
  hunk, expand to the enclosing function and its callers, then judge.
- **Role B — trace-first.** Start from the entry points the change affects
  (routes, commands, handlers, public methods, jobs, migrations, tests) and
  follow the data and control flow into and through the changed code. Then
  sweep the diff for any hunk your traces did not reach.

In the debate both roles are equal: each challenges the other's findings and
defends its own.

## Rounds

The orchestrator stops early when nothing is left to settle.

1. **Round 1 — independent review.** Review blind to your peer. Return
   findings in the R1 format.
2. **Round 2 — cross-examination.** You receive your peer's R1 output
   verbatim, plus any **handoff leads** other topics' reviewers routed to your
   topic. Independently re-verify **every** peer finding against the code and
   give it a verdict. Check every handoff lead: if it holds, report it as a new
   finding (full format); if not, say why. Revise your own list if the peer's
   view exposed a mistake (withdraw) or you found something new while checking
   (new finding, full format).
3. **Round 3 — rebuttal.** You receive your peer's R2 output. Answer every
   dispute and amendment on your findings, and give verdicts on the peer's new
   findings.
4. **Round 4 — closing.** Only for items still open after round 3. You receive
   your peer's R3 output and answer what is addressed to you: for a dispute or
   amendment **you** raised that the author defended or rejected, give your
   final position; for **your** round-2 new findings that the peer disputed or
   amended in round 3, respond as the author. No new findings.

After round 4, anything still disputed is **contested** and goes to independent
adjudication. Do not try to "win" by repetition — adjudication reads the
evidence, not the volume.

### Verdicts

On a peer finding (rounds 2 and 3, and on peer new findings):

- `AGREE` — you re-verified it yourself and it holds. Cite what you checked.
  Agreeing without your own check is forbidden.
- `DISPUTE <ground>` — it does not hold. Ground is one of:
  - `NOT-REAL` — the code does not do that, or a guard exists (cite it).
  - `UNREACHABLE` — no real path leads there (cite the trace).
  - `OUT-OF-SCOPE` — pre-existing and untouched by the change, or another
    topic's (say which topic — the orchestrator re-routes it there, so a real
    issue is never lost to a topic boundary).
  - `BELOW-FLOOR` — real, but below the level's floor (e.g. a nit at Basic).
- `AMEND` — real, but the severity, class, location, or fix is wrong. Give
  the corrected value and why.
- `DUPLICATE <id>` — the same issue as one of yours; name which. Duplicates
  count as independent agreement — make sure it is truly the same issue.

On your own finding when challenged (rounds 3 and 4):

- `DEFEND` — new evidence that answers the specific objection. Restating the
  original claim is not a defense.
- `WITHDRAW` — the objection is right. Say what you missed.
- `ACCEPT` / `REJECT` — to an AMEND, with the reason.

Final position in round 4: `CONCEDE` (the defense convinced you — your
dispute is dropped) or `MAINTAIN` (with the specific reason the defense
fails).

### Debate conduct

- **Argue from evidence only** — file:line, a search and its result, a test
  run, installed library code. Taste is not a ground.
- **Withdrawing a wrong finding is a success**; defending a wrong one is the
  worst outcome of the review. Changing your mind needs a reason, and so does
  holding firm.
- **No horse-trading** — never agree to one finding in exchange for another,
  never escalate severity to win an argument, never agree just to end the
  debate.
- **Answer every item.** An item without a verdict stalls the review — the
  orchestrator will ask again.

## Output formats

Use these exactly; the orchestrator parses them. Finding IDs are
`<TOPIC>-<ROLE><n>` with topic codes `ARC` architecture, `REU` reusability,
`SAF` safety, `SCA` scalability, `SIM` simplicity, `SEC` security, `TST` tests
(e.g. `SAF-A1`, `SEC-B3`). IDs are never reused, even after a withdrawal.

### Round 1

```
ROUND 1 · TOPIC <code> · ROLE <A|B> · LEVEL <hotfix|basic|extended>
FINDINGS <n>

[<ID>] <CRITICAL|HIGH|MEDIUM|NIT> · <failure|quality|nit> · <one-line title>
  Location: <path>:<line>[-<line>]
  Code: <exact quoted line(s)>
  Issue: <what is wrong and why it matters — one or two sentences>
  Evidence:
    - <path:line / search command + result / test run + output> → <what it shows>
    - ...
  Scenario: <input/state/sequence → observable wrong outcome>   (failure class)
  Cost: <the concrete present cost, with references>             (quality/nit)
  Fix: <concrete change, specific enough to act on without a follow-up>

QUESTIONS
  - <location> — <what the author must confirm, and why it matters>   (or: none)
HANDOFF
  - <topic> — <location> — <what to check>                            (or: none)
PRE-EXISTING
  - <location> — <issue>                                              (or: none)
COVERAGE
  Reviewed: <files, or "all changed files">
  Not reviewed / limits: <what and why, or "none">
```

With no findings, write `FINDINGS 0` and still fill QUESTIONS, HANDOFF,
PRE-EXISTING, and COVERAGE.

### Round 2

```
ROUND 2 · TOPIC <code> · ROLE <A|B>
PEER FINDINGS
  <peer ID>: <AGREE|DISPUTE <ground>|AMEND|DUPLICATE <own ID>> — <reason + evidence>
  ...
OWN REVISIONS
  <own ID>: WITHDRAW — <what was wrong>                      (or: none)
HANDOFF LEADS
  <lead ID>: <CONFIRMED → <new ID>|NOT CONFIRMED> — <reason + evidence>   (or: none)
NEW FINDINGS
  <full R1 finding blocks, new IDs>                            (or: none)
```

### Round 3

```
ROUND 3 · TOPIC <code> · ROLE <A|B>
RESPONSES ON MY FINDINGS
  <own ID>: <DEFEND|WITHDRAW|ACCEPT|REJECT> — <reason + new evidence>
  ...
PEER NEW FINDINGS
  <peer ID>: <AGREE|DISPUTE <ground>|AMEND|DUPLICATE <own ID>> — <reason + evidence>   (or: none)
```

### Round 4

```
ROUND 4 · TOPIC <code> · ROLE <A|B>
MY DISPUTES AND AMENDMENTS
  <peer ID>: <CONCEDE|MAINTAIN> — <reason>                   (or: none)
MY NEW FINDINGS
  <own ID>: <DEFEND|WITHDRAW|ACCEPT|REJECT> — <reason + evidence>   (or: none)
```

## Commands — read-only, always

You may: read files; `git diff`, `git log`, `git show`, `git blame`,
`git grep`; `grep`/`rg`/`find`; run the project's **existing** test, lint, or
type-check commands in check mode, scoped to what the change touches, when they
run locally without external services.

Every command must leave `git status --porcelain --untracked-files=all`
exactly as it was. Writes to caches and build output (e.g. `.pytest_cache`,
`var/cache`) are acceptable only when they are git-ignored. Otherwise suppress
them: Python with `PYTHONDONTWRITEBYTECODE=1` (also for one-off `python -c`
probes and imports); other tools via their no-cache flag or a cache directory
in your scratchpad. A probe script lives in your scratchpad, never in the repo.
If a command cannot avoid an unignored write, do not run it; say so under
COVERAGE.

You must not: edit, create, or delete any file in the repository; run anything that
changes the working tree or git state (`checkout`, `switch`, `stash`, `reset`,
`commit`, `rebase`, formatters in write mode); install or upgrade dependencies;
run migrations or anything against a non-test database; make network calls to
external services; touch remote or production environments; read or print the
contents of `.env` files, credentials, or secrets. A committed secret is
reported by location, with the value masked.

If a check needs something you may not do, say so under COVERAGE and treat the
claim as unverified.

## Coverage honesty

Never imply you reviewed what you did not. If the diff was too large to read
fully, if a file could not be opened, or if a test could not be run, say so in
COVERAGE. "No findings" with an honest coverage line is a valid result; "no
findings" from a review that skipped half the diff is a false negative waiting
to ship.
