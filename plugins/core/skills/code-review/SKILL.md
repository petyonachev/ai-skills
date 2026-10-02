---
name: code-review
description: >-
  Interactive, level-calibrated, adversarial multi-agent review of code
  changes — the final gate before production. Invoked as a command
  (/core:code-review [hotfix|basic|extended] [target]). Asks the review level,
  then runs two independent reviewers per topic — Architecture, Reusability,
  Safety, Scalability, Simplicity, Security, Tests — that cross-examine each
  other's findings against the code; contested claims go to adversarial
  verifiers and every surviving claim is re-checked before it is reported.
  Hotfix covers only what can make production fail; Basic covers all topics
  without nits; Extended adds nits. Reports each issue with location, evidence,
  and a concrete fix. Triggers: "/core:code-review", "review my changes", "review
  this code", "review before PR", "check the diff for issues", "review this
  branch", "review PR #123".
---

# Code review

The last gate before code reaches production. It reports **only verified,
actionable issues, calibrated to the level the change needs** — and nothing
else. A false positive wastes the author's time and teaches them to ignore the
review; a false negative ships a defect. This workflow attacks both:

- **Recall** — every topic is reviewed by **two independent reviewers** taking
  different routes through the change (diff-first and trace-first).
- **Precision** — the two **argue every finding** against the code until each
  is agreed, withdrawn, or contested; contested claims go to adversarial
  `verifier` agents; and **you re-check every surviving claim** yourself before
  it reaches the report.

The reviewers are the `review-<topic>` agents; the rules they follow — evidence,
classes and severities, level floors, debate rounds, output formats — are in
`code-review-protocol`. That skill is the contract; this one is the procedure.
You are the **orchestrator**: you build the packet, dispatch the pairs, relay
the debate verbatim, settle the ledger, verify, and report. You never invent a
finding, and you never pass one on unverified.

## The levels

| Level | For | Topics (two reviewers each) | Reports |
|---|---|---|---|
| **Hotfix** | Must ship fast; only prove it cannot break production. No architecture or cleanliness. | Safety, Security, Scalability, Tests | **Failure class only** — anything that can make production fail. |
| **Basic** | Normal PRs. Every topic reviewed thoroughly. | All seven | Failure + quality (Critical/High/Medium). **No nits.** |
| **Extended** | Flagship or long-lived code; ship it clean. | All seven | Everything, **including nits.** |

Topic codes: `ARC` Architecture, `REU` Reusability, `SAF` Safety, `SCA`
Scalability, `SIM` Simplicity, `SEC` Security, `TST` Tests. Agents:
`review-architecture`, `review-reusability`, `review-safety`,
`review-scalability`, `review-simplicity`, `review-security`, `review-tests`.

Legacy level names map as: `quick`, `medium` → **Basic**; `extensive` →
**Extended**.

## 0. Prepare

1. Load `code-review-protocol` with the Skill tool. You parse every reviewer
   output against its formats and apply its class/severity/level rules.
2. Make sure you can resume agents: the `SendMessage` tool. If it is deferred,
   load it (ToolSearch `select:SendMessage`). If it is unavailable, use the
   fallback in step 4.
3. Confirm this is a git repository (`git rev-parse --show-toplevel`). If not,
   stop and say so.

## 1. Scope the change and build the packet

Every reviewer must review the **same snapshot** with the **same context**. You
build that once, as a packet directory.

### Determine the target

From the arguments, else the default:

- **Default (no target)** — the current branch against its base, **including**
  staged, unstaged, and untracked changes. Base: `origin/HEAD`'s branch, else
  `main`, else `master`; `BASE=$(git merge-base HEAD <base-branch>)`. If HEAD is
  on the base branch itself, review only the uncommitted changes
  (`BASE=HEAD`); if there are none, ask what to review.
- **PR number** (`#123`, `123`) — `gh pr view <n> --json
  number,title,body,baseRefName,headRefName,headRefOid`, diff from
  `gh pr diff <n>`.
- **Branch or commit range** (`feature-x`, `A..B`) — diff of the merge base
  (or `A`) to the branch tip (or `B`).
- **Path** (`src/Billing`) — any of the above, limited to `-- <path>`.

If the head under review is **not the current working tree** (a PR or branch
not checked out, a range ending before HEAD), the reviewers would read the
wrong files. Create a detached worktree at that head inside the packet —
`git worktree add --detach <packet>/worktree <head-sha>` (fetch first if the
commit is not local) — and point the reviewers at it. Never check out over the
user's working tree. Remove the worktree in step 9.

### Build the packet

Create a uniquely named packet directory —
`mktemp -d "<scratchpad>/review-packet.XXXXXX"` if the session has a scratchpad
directory, else `mktemp -d` — never a fixed name: concurrent reviews would
overwrite each other's packet. Use that exact path in every brief. Write:

- **`diff.patch`** — the full diff. For the default target: `git diff $BASE`
  (committed + staged + unstaged, tracked files), then append every untracked,
  non-ignored file (`git ls-files --others --exclude-standard`) as
  `git diff --no-index /dev/null <file>` (exit status 1 is normal there).
  Untracked files are the most commonly missed part of a review — include them.
- **`files.txt`** — each changed file with status (A/M/D/R) and added/removed
  line counts (`git diff --numstat` plus the untracked files), then an
  **Excluded** list with reasons: lockfiles, generated or minified files,
  vendored code, binaries. Lockfiles are excluded from line review, but any
  dependency change (manifest or lockfile) is listed under **Dependency
  changes** with package and version.
- **`intent.md`** — what the change is meant to do: anything the user said,
  commit subjects and bodies (`git log --format='%h %s%n%b' $BASE..<head>`),
  the PR title and body if a PR exists (`gh pr view` — skip silently if `gh` is
  unavailable), the ticket id from the branch name. If no intent can be found
  and the session is interactive, ask the user for one sentence in step 2.
- **`meta.md`** — level, active topics, repository root to read post-change
  files from (the repo root or the packet worktree), base and head refs/SHAs,
  the exact command that reproduces the diff, the detected stack (e.g.
  `composer.json` with Symfony → `symfony-stack`; `pyproject.toml` →
  `python-stack`; `*.csproj` → `csharp-stack`), the stack skills available in
  this session, and the project's test command if it is discoverable (README,
  CI config, `Makefile`, manifest scripts).
- **`tree-state.txt`** — `git status --porcelain --untracked-files=all` of the
  repository root reviewers read from, taken before any reviewer is dispatched.
  Step 9 compares against it.

Stop if the diff is empty. If it is very large (above ~3,000 changed lines
excluding exclusions), tell the user before dispatching: a review that cannot
read everything degrades into sampling. Offer to split it by directory or
commit; proceed whole only if they choose to — reviewers will report coverage
limits honestly.

## 2. Ask the level

If the level was given as an argument, use it. Otherwise ask with an
interactive question (AskUserQuestion), combining it with scope confirmation
and the intent sentence if those are open:

- **Hotfix** — must ship fast; only checks what can make production fail
  (Safety, Security, Scalability, Tests). No architecture or cleanliness.
- **Basic** — all seven topics reviewed thoroughly; blocking issues only, no
  nits.
- **Extended** — all seven topics, plus non-blocking nits.

In a non-interactive session with no level given, use **Basic** and state that
in the report.

## 3. Round 1 — dispatch the pairs

For every active topic, spawn **two** agents of that topic's type — role **A**
and role **B** — all in **one batch** so they run in parallel (Hotfix: 8
agents; Basic and Extended: 14). Every active topic gets its pair, even if it
looks irrelevant to the diff: an empty result is cheap and the pair decides
relevance, not you. Record each agent's ID (and name, if your Agent tool
assigns one) against `<topic>-<role>`.

Round 1 brief, exactly this shape — the protocol and rubric are preloaded in
the agent, the packet carries the context:

```
You are the <Topic> reviewer, role <A|B>, round 1 of a code-review debate.
Packet: <packet path> — read meta.md, intent.md, files.txt, and diff.patch first.
Post-change files are under: <repository root or worktree path>.
Level: <hotfix|basic|extended>.
Follow code-review-protocol: your role's route, the evidence rules, the level
floor for your topic, and the Round 1 format. Do not modify anything.
```

Wait for every agent to finish. Never summarize or predict an agent's result
before it arrives.

**Check each output** before moving on:

- Malformed (not in the R1 format): resume the agent once, asking it to
  re-emit in the R1 format.
- `PROTOCOL MISSING`, an error, or no output: spawn a replacement once. If it
  fails again, that topic proceeds with a **single reviewer** — skip its debate,
  send every one of its findings to adjudication (step 6), and state the gap in
  the report.

**Route handoffs.** Collect every HANDOFF line, number them `L1, L2, …`, and
attach each to the owning topic's round-2 messages (both A and B). A lead whose
topic is not active at this level is dropped — a lead that can make production
fail always belongs to an active Hotfix topic (Safety, Security, Scalability,
Tests).

## 4. Rounds 2–4 — the debate

You relay; the reviewers argue. Resume each agent with SendMessage (its ID or
name) so it keeps its context. **Relay outputs verbatim** — never paraphrase,
summarize, or editorialize between them, and never take a side. Run all topics'
rounds in parallel.

- **Round 2 — cross-examination.** To each agent: its peer's full R1 output,
  plus the handoff leads for its topic.

  ```
  Round 2 — cross-examination. Your peer (<Topic> role <X>) returned this
  round-1 output, verbatim:
  <<<
  <peer R1 output>
  >>>
  Handoff leads routed to your topic (verify each; list it under HANDOFF LEADS):
  - L<n> (from <ID or topic-role>): <location> — <what to check>     (or: none)
  Respond in the Round 2 format.
  ```

- **Round 3 — rebuttal.** Only if round 2 left anything to answer: any DISPUTE
  or AMEND on a finding, any NEW FINDINGS (including confirmed leads), or a
  WITHDRAW that removes an item the peer AGREED to as a DUPLICATE. To each
  agent: its peer's full R2 output, verbatim, with "Respond in the Round 3
  format."
- **Round 4 — closing.** Only if round 3 left anything open: a DEFEND or REJECT
  answering a dispute or amendment, or a DISPUTE or AMEND on a round-2 new
  finding. To each agent with open items: its peer's full R3 output, verbatim,
  with "Respond in the Round 4 format."

The debate ends after round 4 at the latest. **Silence is never agreement**: if
an agent leaves an item without a verdict, ask it once for that item; if it
still does not answer, the item is contested.

**Fallback without SendMessage** (or if an agent cannot be resumed): spawn a
fresh agent of the same type and role with the packet path, its own previous
outputs verbatim, the peer output for this round verbatim, and "You are
continuing a code-review debate; re-verify against the code before answering."

## 5. Settle the ledger

Build one ledger per topic from the debate record. Each finding ends in exactly
one state:

| Debate record | State |
|---|---|
| Raised by both (DUPLICATE), or the peer AGREED | **Agreed** |
| AMEND → ACCEPT | **Agreed**, amended |
| AMEND → REJECT → CONCEDE | **Agreed**, original |
| AMEND → REJECT → MAINTAIN | **Agreed** on existence; you settle the disputed attribute from the code and record why |
| DISPUTE → WITHDRAW, or withdrawn by its author at any point | **Dropped** |
| DISPUTE → DEFEND → CONCEDE | **Agreed** |
| DISPUTE → DEFEND → MAINTAIN | **Contested** |
| Round-2 new finding: DISPUTE in R3 → DEFEND in R4 | **Contested** |
| Round-2 new finding: DISPUTE in R3 → WITHDRAW in R4 | **Dropped** |
| No verdict after one re-ask | **Contested** |
| Single-reviewer topic (agent failure) | **Contested** |
| Dropped **only** because it belongs to another topic (`OUT-OF-SCOPE` — other topic) | **Re-routed** — if that topic is active at this level, the finding goes to adjudication (step 6) under the owning topic, since that topic's pair never debated it; it is never lost to a topic boundary |

Keep every dropped finding and its reason for the debate summary — you do not
report them as issues.

## 6. Adjudicate contested findings

Dispatch the `verifier` agent — refute by default — on each contested and
re-routed finding.
**Critical or High: two independent verifiers. Medium or Nit: one.** Brief:

```
Claim: <the finding, verbatim: ID, severity, class, location, code, issue, scenario/cost>
Packet: <path>; post-change files under <root>. Level: <level>.
Reviewer's case: <the author's R1 finding and R3/R4 defense, verbatim>
Challenger's case: <the dispute(s), verbatim>
Decide whether the claim is real, reachable, in scope (introduced or exposed by
this diff), and at or above the level floor: <floor for this level, from the
protocol>. This is a static code claim: a complete, cited trace of the code path
is acceptable evidence when executing it is not feasible; run the existing tests
or a reproduction when you safely can. Do not modify anything.
```

| Verifier result | Outcome |
|---|---|
| All CONFIRMED | **Confirmed** — reported, marked "confirmed on adjudication" |
| All REFUTED | **Dropped** |
| Split, or any INCONCLUSIVE | **Needs author confirmation** — reported in its own section, not as an issue |

## 7. Verify every surviving claim yourself

Agreement between two agents is strong evidence, not proof. For **every**
agreed or confirmed finding, open the code and check:

1. **Location** — the cited lines exist and the `Code` quote matches them
   exactly. A slightly wrong line number: correct it.
2. **Scope** — the code is in the diff, or the diff changes its behavior.
3. **Substance** — the core claim holds when you read it: the guard is really
   absent (repeat the decisive search), the caller path really reaches it, the
   cited existing code really exists and matches.
4. **Severity and level** — severity follows the protocol rubric on the
   defect's own impact, not the level; class is at or above this level's floor.

If a substance or scope check fails and you cannot resolve it by reading,
dispatch one `verifier` with the step-6 brief: REFUTED drops it, CONFIRMED keeps
it, INCONCLUSIVE moves it to "Needs author confirmation". Never drop a finding
silently and never keep one you could not check.

## 8. Merge, filter, sort

- **Merge across topics** — the same root cause at the same location raised by
  different topics becomes one issue: keep the highest severity that a single
  topic's finding justifies on its own (merging never escalates it), list every
  topic, combine the evidence.
- **Apply the level floor** once more — Hotfix: failure class only; Basic: no
  nits; Extended: everything.
- **Sort** by severity (Critical, High, Medium, Nit), then by file and line.
- **Questions** — collect reviewers' QUESTIONS, dedupe, and drop any you can
  answer from the code yourself (answering them is better than asking). Keep
  only those relevant to this level.
- **Pre-existing** — collect PRE-EXISTING notes, dedupe, keep at most five, only
  serious ones.
- **Reconcile** — before writing the report, map every Agreed or Confirmed
  ledger entry to exactly one reported issue, standalone or merged into a named
  one (a level-floor drop is recorded as such, never silent). Compute the review
  record's per-topic counts from the issues you list, not from memory. Every
  issue rated by a calibration row that requires a question to the author has
  that question under Questions. A ledger entry with no issue, a count that
  does not match, or a missing required question is an error: resolve it before
  reporting.

## 9. Report

```
## Code review — <Level> · <scope: base..head, N files, +A/−D>
Verdict: <Ship | Fix first | Blocked> — <one-line rationale>
Critical N · High N · Medium N · Nit N        (Nit only at Extended)

### Critical
[CRITICAL] <title> — <topic(s)> · <failure|quality>
  Location: <path:line>
  Issue: <what is wrong and why it matters>
  Evidence: <the decisive evidence — trace, search, test run, existing code>
  Scenario / Cost: <concrete failure scenario, or concrete present cost>
  Fix: <concrete change>
  Verified: <agreed by both reviewers | confirmed on adjudication>

### High / Medium / Nit
...

### Needs author confirmation
- <path:line> — <claim> — <why it could not be settled; what would settle it>

### Questions for the author
- <path:line> — <question>

### Pre-existing (not introduced by this change; not counted)
- <path:line> — <issue>

### Review record
<Topic>: <N reported> · <N dropped in debate> · <N refuted on adjudication> · coverage <full | limits>
...
Debate: <N raised → N reported>; reviewer failures or coverage gaps: <list, or none>
```

Omit an empty section, except **Review record**, which is always present: it
proves each topic was actually reviewed, and a topic with no findings shows
`0 reported`. Keep every issue self-contained — what, where, why, and how to
fix, without a follow-up question.

**Verdict rules:**

- **Blocked** — any Critical issue.
- **Fix first** — any High or Medium issue, or any Critical/High item under
  "Needs author confirmation".
- **Ship** — nothing above, only nits (Extended) or nothing at all. A Ship
  verdict with coverage gaps says so in its rationale.

**Clean up**: re-run `git status --porcelain --untracked-files=all` and diff it
against `tree-state.txt`. Any difference is a reviewer side effect: list each
path in the report and offer to remove or restore it — never delete or restore
without the user's approval. Then remove a packet worktree you created (`git
worktree remove <path>`). Leave the rest of the packet in the scratchpad.

## Anti-patterns

- **Reporting unverified claims** — every reported issue passed the debate or
  adjudication and your step-7 check. No exceptions at any level.
- **Agreement by assumption** — treating silence, a missing verdict, or "looks
  right" as agreement.
- **Editorial relay** — paraphrasing or summarizing one reviewer to the other;
  relay verbatim.
- **Orchestrator findings** — adding issues no reviewer raised. If you notice
  something, route it as a handoff lead to the owning topic.
- **Wrong strictness** — architecture or style findings in a Hotfix; nits in a
  Basic review; waving issues through at Extended.
- **Scope creep** — pre-existing problems reported as if introduced.
- **Silent coverage gaps** — a failed reviewer or a skipped part of the diff
  reported as "no issues".
- **Reviewing the wrong snapshot** — reading the working tree while reviewing a
  PR head that is not checked out.

## Where this fits

`code-review` is a Layer 1 workflow and the final gate before `finish-branch`.
It runs the `review-<topic>` agents under `code-review-protocol`, which apply
the criteria skills (`architecture`, `solid`, `engineering-standards`,
`code-quality`, `refactoring-catalog`, `security`, `performance`,
`database-design`, `integration-build`, `testing`, `verify`, plus the stack
skills), and settles contested claims with the `verifier` agent
(`parallel-agents` adversarial-verify pattern). It is the deeper, external pass
after `feature-delivery`'s self-review.
