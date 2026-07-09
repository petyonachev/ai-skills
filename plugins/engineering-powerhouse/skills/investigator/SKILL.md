---
name: investigator
description: >-
  Conduct a bug investigation AND document it as a polished, standalone HTML
  report ready to open in a browser. Use when investigating a bug or incident and
  a written, shareable record is wanted — tracking evidence from code and data
  (files, queries, logs), then producing a styled report covering problem,
  severity/impact, reproduction, investigation timeline, evidence, root-cause
  diagnosis, fix, and prevention. Runs the debug method; owns the tracking and the
  report artifact. Triggers: "investigate this bug", "produce an investigation
  report", "root cause report", "document this incident", "bug investigation
  writeup", "post-mortem".
---

# Investigator

This skill does two things: it *runs* a disciplined bug investigation, and it
*documents* it as a standalone HTML report someone can open and read without you in
the room. The investigation method itself is `debug` (reproduce → evidence →
hypothesize → root cause → fix → prevent); this skill adds two things `debug` does
not: a structured **evidence log** you maintain as you go, and a **report artifact**
at the end.

Track as you investigate — reconstructing evidence afterwards loses detail and
invites hindsight bias. The report is assembled from the log, not from memory.

## What to track (the evidence log)

Capture these as you work; they become the report's sections:

- **Problem** — the symptom as observed: what is wrong, who reported it, when it
  started.
- **Severity & impact** — users affected, data/financial risk, frequency,
  workaround availability. Classify it (below).
- **Environment** — version/commit, environment (prod/staging), config, and the
  data conditions under which it occurs.
- **Reproduction** — the minimal reliable steps that trigger it (or why it could
  not be reproduced, and what that implies).
- **Investigation timeline** — the ordered trail of what you checked and what it
  told you, including dead ends. This is the "steps done" narrative.
- **Evidence** — concrete artifacts, each cited: code references (`file:line`),
  queries run **and their results**, log excerpts, data findings. Redact secrets
  and PII (`security`).
- **Diagnosis** — the root cause via the why-chain (`debug`): not the first
  plausible line, the actual cause. State it plainly and explain *why* it produces
  the symptom.
- **Fix** — proposed or applied, with its verification (`verify`: reproduction gone,
  regression test added).
- **Prevention** — how to stop the class of bug recurring; follow-ups and
  recommendations.

## Severity classification

Assign one, with a one-line justification:

| Severity | Criteria |
|---|---|
| **Critical** | Data loss/corruption, security breach, or core flow down in production; no workaround. |
| **High** | Major feature broken or significant user impact; workaround painful or partial. |
| **Medium** | Limited or intermittent impact; a reasonable workaround exists. |
| **Low** | Cosmetic or edge-case; minimal impact. |

## The report artifact

Produce a **single standalone HTML file** (e.g. `investigation-<slug>.html`) written
to disk, ready to open in a browser:

- **Self-contained** — all CSS inline in a `<style>` block; **no external
  stylesheets, fonts, scripts, or images**. It must render offline from one file.
- **Structured** — one section per evidence-log item above, in that order, with a
  metadata header (title, date, author, affected version) and a color-coded
  **severity badge**.
- **Readable** — clean professional styling, generous whitespace, monospace `<pre>`
  blocks for code/queries/logs, a light theme, and print-friendliness.

Use `references/report-template.html` as the starting point — copy it, fill each
section from the evidence log, and save the result. Do not invent a new layout each
time; adapt the template so reports are consistent.

## Anti-patterns

- **No root cause** — a report that describes the symptom and the fix but never
  says *why* it happened (`debug`: that is not a diagnosis).
- **Unverified fix** — claiming "fixed" without the reproduction-gone evidence
  (`verify`).
- **Missing severity/impact** — a reader cannot triage without it.
- **Uncited evidence** — assertions with no `file:line`, query, or log to back them.
- **External dependencies in the HTML** — CDN CSS/fonts that break offline.
- **Leaked secrets/PII** — dumping raw logs or query results containing sensitive
  data.

## Where this fits

`investigator` is the Layer 5 documentation counterpart to `debug`: `debug` finds
and fixes, `investigator` records the finding as a shareable artifact. It relies on
`verify` for the fix evidence, `security` for redaction, and — when the bug is
performance- or data-related — `performance` and `database-design` for the analysis
it documents.
