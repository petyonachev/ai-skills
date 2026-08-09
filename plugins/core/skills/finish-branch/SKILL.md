---
name: finish-branch
description: >-
  End-to-end workflow for shipping a finished branch: run the pre-ship
  verification gate, review the whole diff, choose the integration path (direct
  merge, pull request, squash, rebase), write the commit and PR message, push
  safely, and clean up branches and worktrees. Use when implementation is
  complete and the branch is ready to ship. Covers rebase-vs-merge-vs-squash,
  push safety, and recovery from common missteps. Triggers: "I'm done", "ready
  to merge", "ready to push", "ready for review", "wrap up this branch", "finish
  this up", "ship it".
---

# Finish branch

Finishing is not "the code works" — it is the code verified, the diff reviewed,
the history clean, the message written, and the branch integrated safely. This
workflow runs the last mile so that "done" means shippable, not hopeful.

**Commit and push only when the user asks. If you are on the default branch
(`main`/`master`), create a feature branch first — never commit work directly to
the default branch.**

## 1. The pre-ship gate

Do not proceed until the work is genuinely verified. Hand off to `verify`: full
test suite, lint, type-check, and build all green — run, not assumed — and the
actual behavior driven end-to-end. A branch that has not cleared `verify` is not
finished.

## 2. Review the whole diff

Read every hunk of the branch as a reviewer, not just your last change. A branch
accumulates more than you remember:

- Does the whole diff match the intended scope? Nothing unrelated snuck in?
- No secrets, credentials, or `.env` values committed.
- No leftover debug output, commented-out code, or dead code.
- No stray files (build artifacts, editor configs, scratch files).
- Conventions and layering consistent across the branch.

`git diff main...HEAD` and read it top to bottom. Fix what you find before
shipping. For a large or high-risk branch, dispatch an independent reviewer agent
(`parallel-agents`).

## 3. Choose the integration path

Match the strategy to the branch:

- **Rebase vs. merge** — rebase your feature branch onto the latest `main` to keep
  a linear, clean history *before* integrating. **Never rebase a branch others
  have pulled** — rewriting shared history breaks everyone downstream.
- **Squash vs. preserve commits** — squash when the branch is messy WIP that
  represents one logical change. Preserve commits when each is meaningful and
  tells the story of the change.
- **PR vs. direct merge** — open a PR when the change needs review or a record
  (the default for team work). Direct-merge only for trivial, solo, low-risk
  changes where review adds nothing.

### Calibration — worked examples

| Situation | Path |
|---|---|
| Solo trivial fix, low risk | Fast-forward / direct merge; skip the PR. |
| Team feature needing review | PR; squash or preserve per commit quality. |
| Messy WIP commits, one logical change | Squash into one clean commit. |
| Several meaningful, self-contained commits | Merge (or rebase) preserving them. |
| Feature branch fallen behind `main` | Rebase onto `main` if unshared; merge `main` in if shared. |
| Stacked branches | Integrate bottom-up, in order; rebase each on its parent. |

The pattern: **clean, linear history and one PR per logical change; preserve
commits only when they each carry real meaning; never rewrite shared history.**

## 4. Commit and PR conventions

These are the carried-forward house defaults — adjust in `engineering-standards`
if they change:

- **Branch name**: ticket prefix, kebab-case — `PROJ-123-name-of-the-branch`.
- **Commit subject**: ticket prefix (no colon), imperative mood, under 72
  characters — `PROJ-123 add order confirmation webhook`. Use the body for the
  *why* on non-trivial changes.
- **Do not add the agent as a co-author.**
- **Clean up WIP** before pushing — no `wip`, `fix`, `asdf` commit messages in the
  final history.
- **PR**: title under 72 characters; description with `## Summary` and `## Test
  Plan`; link the related ticket/issue. Defer the full write-up to `pr-writer`.

## 5. Push safely

- **Never force-push `main`/`master`** — no exceptions.
- **Never force-push a shared branch** others may have pulled.
- Use `--force-with-lease`, never bare `--force`, and only on your own branch
  after a rebase — it refuses to clobber commits you have not seen.
- Push the branch, then open the PR; confirm CI is green before asking for review.

## 6. Clean up

- Delete the branch after it is merged — but only with approval, and never an
  *unmerged* branch (you would lose the work).
- Remove the worktree if you used one, after confirming no uncommitted changes and
  nothing unpushed.
- Do not modify CI/CD pipelines as part of finishing unless that was the task and
  it was approved.

## Recovery from common missteps

- **Committed to `main` by mistake** — branch from the current state, then reset
  `main` back (`git reset --hard origin/main`) *only* if `main` is unpushed;
  if pushed, revert instead.
- **Force-pushed the wrong branch** — recover the overwritten commits from
  `git reflog` and restore them; this is exactly why `--force-with-lease` exists.
- **Deleted an unmerged branch** — the commits usually survive in `git reflog`;
  recreate the branch pointing at the lost tip before it is garbage-collected.
- **Rebased a shared branch** — coordinate with everyone who pulled it; they must
  reset to the new history. Avoid by never rebasing shared branches.

## Anti-patterns

- **Shipping unverified** — "finishing" before the `verify` gate passes.
- **Blind push** — not reading the full diff before it goes out.
- **Force-pushing `main`** or any shared branch.
- **Kitchen-sink branch** — unrelated changes bundled into one PR.
- **WIP history** — meaningless commit messages in the permanent record.
- **Deleting unmerged work** — removing a branch or worktree with unshipped
  commits.

## Where this fits

`finish-branch` is the closing Layer 1 workflow — every other flagship
(`feature-delivery`, `refactor-safely`, `debug`, `integration-build`) ends here.
It runs the `verify` gate, uses `parallel-agents` for independent final review,
hands the description to `pr-writer`, and draws its naming and message conventions
from `engineering-standards`.
