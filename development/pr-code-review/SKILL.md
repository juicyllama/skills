---
name: pr-code-review
description: Review the code an open PR proposes — correctness, security, performance, maintainability — combining your own reading of the diff with the static-analysis tooling War Room ran for you, and publish the result as inline PR review threads the War Room review loop can act on. Runs in full mode for the first review of a PR and incremental mode for every push after it.
---

Canonical, adapter-agnostic procedure for an **automated code review** of a branch submitted as a PR. Any agent — Claude, Codex, or a human — follows this. Per-adapter entry points import it:

- **Codex / generic agents:** referenced from [`AGENTS.md`](../AGENTS.md).
- **Claude Code:** the `pr-code-review` skill points here.

This is War Room's own reviewer. It replaces the third-party review bot the loop used to wait on, and it is deliberately shaped to produce the *same artefact* that bot produced — inline review threads, each carrying a severity, an explanation, a committable fix where one exists, and an agent-executable instruction — because the whole downstream machinery (thread replies, resolution, the merge gate) reads that artefact.

## What this is, and what it is not

This answers one question: **is the code this PR proposes correct, safe, and fit to merge?**

- It is **not** a scope review. Whether the PR delivers what the issue asked for is the `completeness-check` skill's job. Do not build a requirements checklist here, and do not flag "the issue also asked for X" — that gate runs separately and owns it.
- It is **not** a refactor pass. Broad de-duplication and simplification belong to the `dry` skill. You may flag a *local* duplication or complexity problem inside the diff as a nitpick; you never restructure the branch.
- It is **not** an implementation session. You review and report. You do not edit source files, do not commit, and do not push. A separate remediation pass acts on your threads.
- It **is** a correctness, security, performance, and maintainability review of the lines this PR actually changed.

## The two modes

War Room tells you which mode you are in. They differ in **scope**, not in standard.

### `full` — the first review of the PR

The review scope is the PR's entire diff against its base: `git diff <base>...HEAD`. Every changed hunk is in scope, and you also publish the walkthrough summary comment (see step 7) that describes the PR as a whole.

### `incremental` — every push after a review already landed

The review scope is only what changed **since the last reviewed commit**: `git diff <lastReviewedSha>..HEAD`. War Room hands you that sha.

Incremental mode is where reviews usually go wrong, so it has its own rules:

- **Review the new commits, but judge them against the whole PR.** A hunk added in the latest push may be individually fine and still break something the earlier part of the PR set up. Read the surrounding code and the full-PR diff for *context*; raise findings only on lines inside the incremental range.
- **Never re-raise a finding you already published.** War Room hands you the threads you opened on earlier passes. If the code still has the problem, the existing thread is still open and still carries it — opening a second thread for the same defect is noise that a human then has to close twice.
- **Do re-raise a finding that was closed and regressed.** If a thread was resolved and the latest push reintroduces the defect, open a fresh finding and say explicitly that it regressed.
- **Check the fixes.** For each thread that was resolved since your last pass, confirm the current code actually fixes it. A fix that silences the symptom (a widened type, a swallowed error, a deleted assertion) is a new finding, and it is a `⚠️ Potential issue`, not a nitpick.
- **Say when there is nothing.** A push that introduces no problems is a real, common, valuable result. Publish zero findings and say so. Do not manufacture a nitpick to look useful.

## Non-negotiables

- **Every finding is anchored to a changed line.** File path, start line, end line, on a line this PR touched. A finding you cannot anchor is not publishable — put it in the walkthrough's notes instead.
- **Review the diff, not the repository.** Pre-existing problems in untouched code are out of scope, however tempting. The one exception: the diff *makes an existing latent bug reachable*. Then it is in scope, and say why the diff is what surfaced it.
- **No speculation presented as fact.** If a finding depends on something you could not verify — a runtime value, an external contract, a call site you could not find — mark it `💡 Verification agent` and state exactly what would confirm or kill it. Never dress an unverified guess as a defect.
- **Severity is a promise, not a flourish.** `🔴 Critical` means it will cause data loss, a security breach, or a production outage. Inflating severity trains everyone to ignore it. Under-calling a security bug is worse. Calibrate honestly against the rubric below.
- **Tool findings are evidence, not verdicts.** Every finding a linter or scanner reports is triaged by you before it reaches the PR. Suppress the false positives, merge the duplicates, and keep the real ones — with the tool named as the source.
- **A suggestion must apply cleanly.** If you attach a committable suggestion block, it replaces the exact line range you anchored, is complete, and compiles in context. A suggestion that does not apply is worse than prose, because someone will click it.
- **Report, do not implement.** No source edits, no commits, no pushes. Your entire output is review threads, one summary comment, and the machine-readable file.

## Process

### 1. Establish the scope

- Read the context War Room handed you: the PR, its base and head shas, the mode, and (in incremental mode) the last reviewed sha and your previously published threads.
- Produce the diff for your mode. `git diff --name-only` over the same range gives you the changed file list you will need in step 2.
- Read the PR description and the linked issue **for context only** — enough to know what the code is trying to do. You are not grading it against them.

### 2. Choose the static-analysis tooling

Before reviewing anything, decide which analysers are worth running against **this** codebase and **these** changed files, and emit that decision as a JSON array. War Room executes the tools you name and hands the results back to you in step 3.

The full catalogue — every tool, what it reviews, and what makes it applicable — is in [`tools.md`](tools.md). Read it and select from it.

How to select:

- **Match the changed files, not the repo.** A repo containing Go does not warrant `golangci-lint` on a PR that only touched Markdown. Select a tool when the PR changed files it actually analyses.
- **Always consider the cross-cutting three.** Secret scanning, dependency/vulnerability scanning, and semantic pattern matching apply to almost every PR regardless of language, because the thing they catch does not care what language it is written in.
- **Prefer the tool the repo already configured.** If a config file exists (`.eslintrc*`, `biome.json`, `ruff.toml`, `.golangci.yml`, `.rubocop.yml`, …), that tool is the house choice — select it. Where two tools overlap and only one is configured, select the configured one and skip the other.
- **Do not select overlapping linters speculatively.** ESLint *and* Biome *and* Oxlint on the same TypeScript diff produces three copies of the same finding and a large bill. Pick the configured one.
- **Selecting an uninstalled tool is free and harmless.** War Room probes each tool and skips the ones that are not installed, reporting which. Select on relevance; let availability be War Room's problem.

Emit the selection to the tool-selection path War Room gave you, as exactly this shape:

```json
{
  "tools": ["eslint", "semgrep", "trufflehog", "osv-scanner"],
  "rationale": {
    "eslint": "eslint.config.js at the repo root; the PR changes 11 .ts files",
    "semgrep": "cross-cutting; the PR touches auth middleware",
    "trufflehog": "cross-cutting secret scan; the PR adds a .env.example and a config loader",
    "osv-scanner": "pnpm-lock.yaml changed"
  },
  "skipped": {
    "biome": "eslint is the configured linter here; running both would duplicate every finding",
    "rubocop": "no Ruby in this repository"
  }
}
```

`tools` must contain only ids from [`tools.md`](tools.md). `rationale` explains each selection in one line; `skipped` records the near-misses you deliberately declined, so a human can tell the difference between "considered and rejected" and "never thought of it".

If War Room did not give you a tool-selection path, it is running you in review-only mode — skip straight to step 3 with whatever findings it handed you.

### 3. Read the tool findings

War Room runs your selection and hands back the normalised findings: file, line, severity, rule id, message, and the tool that produced it. It also tells you which tools it could not run and why.

Triage every one of them **before** you write your own review:

- **Real and in scope** → it becomes a finding, with the tool named as its source. You still write the explanation; a rule id and a one-line message is not a review.
- **Real but pre-existing** → drop it. The tool scanned whole files; your scope is the diff.
- **False positive** → drop it silently. Do not publish "the linter said X but it is wrong" — nobody needs that thread.
- **Duplicate of another tool's finding** → merge into one, and name both sources.
- **Style-only, and the repo's formatter owns it** → drop it. Formatting is the formatter's job, not a review thread.

A tool that War Room could not run is not a silent gap: note it in the walkthrough so a human knows that coverage was missing on this pass.

### 4. Read the diff yourself

The tools find what they have rules for. This step finds everything else, and it is the part that makes the review worth reading. Walk the diff hunk by hunk and look for:

- **Correctness** — off-by-one, inverted conditions, wrong operator, wrong variable, unhandled branch, incorrect nullish/falsy handling, mishandled empty collections, broken early returns.
- **Contract breaks** — a changed signature, return shape, serialized format, error type, or status code that a caller still expects in the old shape. Hunt the call sites; do not assume.
- **Concurrency and ordering** — races, unawaited promises, missing `await`, shared mutable state, non-atomic read-modify-write, ordering assumptions that only hold today.
- **Error handling** — swallowed exceptions, errors logged and continued past, `catch` blocks that lose context, failure paths that leave state half-written, retries without bounds.
- **Resource handling** — leaked handles/connections/listeners/timers, missing cleanup on the error path, unbounded growth.
- **Security** — injection (SQL, command, path, template), missing authz on a new route or handler, secrets or tokens in code/logs/errors, unsafe deserialisation, SSRF, weak crypto, PII in logs, permissive CORS, missing rate limits on a new public endpoint.
- **Performance** — N+1 queries, work inside a loop that belongs outside it, unbounded fetches, missing index implied by a new query, accidental quadratic behaviour, blocking I/O on a hot path.
- **Data and migrations** — destructive or non-reversible migrations, a migration that locks a large table, a schema change without the code change that needs it (or the reverse), a default that rewrites an existing table.
- **Tests** — new behaviour with no test; a test that asserts the implementation rather than the behaviour; a test weakened or deleted to go green.
- **Maintainability, locally** — a name that misleads, a comment that now contradicts the code, dead code the diff introduced, duplication *within* the diff. Keep these to nitpicks.

Read the code *around* each hunk. Most real defects live in the interaction between the new lines and the lines already there, which is exactly what a diff-only reading cannot see.

### 5. Calibrate severity

| Severity | Means | Examples |
|---|---|---|
| `🔴 Critical` | Data loss, a security breach, or a production outage if merged. | Auth check removed, secret committed, destructive migration, injection in a reachable path. |
| `🟠 Major` | A real defect that will produce wrong behaviour or a broken contract, but is not catastrophic. | Inverted condition, unhandled error path, N+1 on a hot endpoint, broken caller contract. |
| `🟡 Minor` | Correct today, but fragile, unclear, or untested. | Missing test for a new branch, unbounded retry, a misleading name on a public helper. |
| `🔵 Nitpick` | Cosmetic or preference. Never blocks. | Local duplication, a stale comment, an awkward but working construct. |

Two calibration rules that matter more than the table: a finding you cannot fully verify is `💡 Verification agent` regardless of how bad it would be if true, and a security finding is never downgraded because it is "unlikely to be exploited".

### 6. Compose the findings

You **emit** findings; **War Room posts them**. Do not post review comments yourself, and do not create a review — a duplicate review is worse than no review.

That split is deliberate. GitHub rejects an entire review if any one comment anchors to a line outside the diff, so a hand-posted batch loses everything to one bad line number; and every thread must carry a marker the rest of the PR machinery keys off. War Room validates each anchor against the real diff, renders the body in the format below, retries individually if GitHub still refuses, and demotes anything unanchorable to a conversation comment so it is never silently lost.

Your job is the content. For each finding you supply `title`, `body`, `suggestion`, and `agentPrompt` (the exact fields are in step 8), and War Room renders them into this:

```markdown
_⚠️ Potential issue_ | _🟠 Major_

**Retry loop never terminates when the upstream returns 500.**

`fetchWithRetry` decrements `attempts` only on a thrown error, but a 500 resolves
rather than throwing, so the loop re-enters with `attempts` unchanged and spins
until the request timeout kills the process. Every 5xx from `billing-api` becomes
a hung worker.

```suggestion
    if (!response.ok) {
      attempts -= 1;
      continue;
    }
```

<details>
<summary>🤖 Prompt for AI Agents</summary>

```
In src/lib/fetch-with-retry.ts around lines 34 to 41, the retry loop only
decrements `attempts` inside the catch block, so a resolved-but-failed response
(status >= 400) loops forever. Decrement `attempts` on any non-ok response
before continuing, and throw once it reaches zero so the caller sees the failure.
```

</details>

<!-- warroom-review:finding -->
```

The parts War Room assembles from your fields, and what each one demands of you:

1. **The tag line** — from your `kind` and `severity`. Kinds: `potential-issue`, `refactor`, `security`, `performance`, `nitpick`, `verification`. Severities as in the table above.
2. **`title`** — one line stating the defect, not the topic. "Retry loop never terminates on 500" — not "Retry handling".
3. **`body`** — what is wrong, why it matters, and what it causes at runtime. Two to five lines. Name the symbol and the mechanism; a reader must be able to confirm it without you.
4. **The source line** — rendered automatically from `sources` and `ruleId` when a tool found it. Just list the tool in `sources`; do not write the line yourself.
5. **`suggestion`** — a complete, correct replacement for the exact anchored range, or `null`. Omit it rather than guess: a suggestion that does not apply is worse than prose, because someone will click it. Never a partial line, never an ellipsis.
6. **`agentPrompt`** — required on every finding. The instruction a coding agent executes to fix it without reading your prose: name the file, the line range, the defect, and the concrete change, as an imperative addressed to that agent. It must stand alone — an agent that sees only this text must be able to do the work.
7. **The marker** — appended by War Room. You never write it.

One defect per finding. Never batch several into one entry, or the loop cannot resolve them independently. War Room orders them by severity, then file, so do not worry about ordering.

**Anchoring decides whether a finding survives.** `startLine` and `endLine` must be lines this PR actually changed, numbered in the *new* file. A finding anchored to unchanged context cannot become an inline thread and gets demoted to a plain comment, where it carries far less weight. If the defect is genuinely in the interaction with unchanged code, anchor it to the changed line that causes it and explain the rest in the body.

### 7. Supply the walkthrough (full mode only)

In `full` mode, supply a `walkthrough` summarising the change as a whole. War Room posts it as one PR conversation comment, appending the coverage and findings counts itself:

```markdown
## Walkthrough

Two to four sentences describing what this PR changes and how, written for someone
who has not read the diff.

## Changes

| File(s) | Summary |
|---|---|
| `src/lib/fetch-with-retry.ts` | Adds bounded retry with backoff around the billing client. |
| `tests/fetch-with-retry.test.ts` | Covers the retry bound and the 5xx path. |
```

Stop there. The **Review coverage** and **Findings** sections are appended by War Room from what it actually ran — writing them yourself only risks contradicting it.

In `incremental` mode set `walkthrough` to `null` unless something about the push genuinely needs narrating. War Room already publishes the reviewed range and the counts.

### 8. Write the machine-readable result

Write the review to the structured output path War Room gave you, **and** make the same JSON object the entire content of your final message. This file is the deliverable — it is what War Room posts from.

```json
{
  "walkthrough": "## Walkthrough\n\n…markdown, or null in incremental mode…",
  "findings": [
    {
      "file": "src/lib/fetch-with-retry.ts",
      "startLine": 34,
      "endLine": 41,
      "kind": "potential-issue",
      "severity": "major",
      "title": "Retry loop never terminates when the upstream returns 500.",
      "body": "`fetchWithRetry` decrements `attempts` only on a thrown error, but a 500 resolves rather than throwing, so the loop re-enters with `attempts` unchanged and spins until the request timeout kills the process.",
      "suggestion": "    if (!response.ok) {\n      attempts -= 1;\n      continue;\n    }",
      "agentPrompt": "In src/lib/fetch-with-retry.ts around lines 34 to 41, the retry loop only decrements `attempts` inside the catch block, so a resolved-but-failed response (status >= 400) loops forever. Decrement `attempts` on any non-ok response before continuing, and throw once it reaches zero so the caller sees the failure.",
      "sources": ["llm"],
      "ruleId": null
    }
  ]
}
```

`kind` is one of `potential-issue`, `refactor`, `security`, `performance`, `nitpick`, `verification`. `severity` is one of `critical`, `major`, `minor`, `nitpick`. `sources` lists `llm` and/or the tool ids that produced the finding — name every tool that contributed when you merged duplicates.

An empty `findings` array is a valid, expected, and frequently correct result. Never pad it.

### 9. Hand off

Stop. Do not fix what you found, do not post or resolve threads, and do not approve or reject the PR. War Room's review loop takes it from here: it posts your findings, hands each resulting thread to a remediation session, and the merge gate blocks until every one is answered.
