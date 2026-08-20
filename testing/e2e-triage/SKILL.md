---
name: e2e-triage
description: Triage a failing review-phase Playwright e2e run. Work through the steps to determine whether the failure is a regression, an intended behaviour change, or an environmental flake. Report your findings back to the PR and ensure the merge gate is respected.
---

Canonical, adapter-agnostic procedure for triaging a failing **review-phase Playwright e2e** run. Any agent — Claude, Codex, or a human — follows this. Per-adapter entry points import it:

- **Codex / generic agents:** referenced from [`AGENTS.md`](../AGENTS.md).
- **Claude Code:** use the repo-local `e2e-triage` skill entry point.
- **Any runtime:** the `warroom pr review` CLI prints a triage directive pointing here when the run fails.

## When this runs

Two gates lead here, and both stay shut until triage is resolved:

- **The review-phase e2e gate.** `warroom pr review` runs the demo Playwright e2e after code review and **before** offering to merge. On failure it posts a `❌ Playwright e2e — FAILED` comment to the PR and **blocks the merge**.
- **A rejecting commit hook.** The review loop's own commit runs the child repo's `pre-commit` hook (often a full build+test+e2e). When that hook rejects the commit, War Room posts a `❌ pre-commit hook — FAILED` comment to the PR and triages here before retrying the commit. The review fix is **not** on the branch until the hook passes. Leave your fix **uncommitted** — War Room stages it and retries the commit, which re-runs the hook. A failure that was environmental needs no change at all: the retry re-runs the hook and a flake clears on its own.

## Non-negotiables

- **Do not merge** while any e2e test is failing. Never bypass a red gate with `--admin` or by skipping e2e.
- **Never bypass a commit hook** with `--no-verify`. The hook is the gate; healing means it passed, not that we went around it.
- **Never edit a test just to make it green.** A test changes only when the behaviour it asserts changed *on purpose*, and that change is defensible and flagged for a human.
- **Fix the root cause, not the symptom.** Match the surrounding code's conventions.
- **Post your conclusion** (per failure) back to the PR so the human has the record.

## Process

### 1. Gather evidence

- Read the PR's `Playwright e2e — FAILED` comment (the output tail) and the failing test names.
- For each failing spec, open the test file and `test-results/<...>/error-context.md` (page snapshot + error), and the `trace.zip` if present.
- Identify the exact failing step (navigation, challenge, assertion, timeout) — not just "it timed out".
- `git diff` the branch against its base for the code paths each failing test exercises. Failures are almost always explained by what changed here.

### 2. Rule out environmental flakiness FIRST

Before blaming code or test, confirm the failure is real. On a shared/local dev stack the e2e can fail for reasons that are neither a bug nor a test defect:

- **Machine oversubscription** — too many workers on a box also running other dev stacks; heavy tests (3DS, iframes) blow the per-test timeout, the failing set changes run-to-run, wall-clock balloons. Check load average; let it settle (< ~4); don't run back-to-back.
- **External-service throttling** — repeated full runs can throttle a shared Stripe/PayPal **test account**, after which flows stop resolving server-side (e.g. 3DS sessions stuck `ACTION_REQUIRED`/`pending`). Symptom: ALL tests of one external-dependent kind fail together in a full run but pass in isolation.
- **Cold-compile / first-hit latency** under `next dev` when the worker pool hits a route cold.

**Test for it:** re-run the specific failing test(s) **in isolation, single worker, on a settled machine**, wrapped in the workspace e2e gate so your run queues instead of stampeding a suite another session is timing: `warroom e2e-exec --label "<pr> e2e-triage" -- env PLAYWRIGHT_WORKERS=1 playwright test <spec> -g "<name>"`. The wrapper waits for the e2e slot AND for box pressure (load, live dev-stack count) and prints "been waiting Xm" notices while it does — a long wait means the box is busy, not that your run is broken, and "settled machine" is exactly what it is waiting for. If the test passes alone, the full-run failure was environmental — do **not** change code or tests. Instead: note it on the PR, and re-run `warroom pr review` when the environment is fresh (lower `WARROOM_E2E_WORKERS`, or `repos.yaml defaults.e2e_workers`, if the box is small).

### 3. Ensure tests are asserting outcomes, not mechanics

E2E tests must assert the **user-visible outcome** of a flow, not the internal mechanics the SDK or backend happens to use to get there. Implementation details (which endpoints get called, in what order, with what payloads) change over time; the product outcomes do not — and tests pinned to mechanics break on refactors that didn't break the product.

**Wrong** — asserting how the code works internally:

```ts
expect(call, 'SDK never called POST …/3ds/complete after the 3DS challenge').not.toBeNull()
```

**Correct** — asserting what the user experiences:

- The user reaches the success page.
- The user sees the decline message.
- The saved payment method appears on the next checkout.

Guidelines:

- Prefer assertions on rendered UI state (success page, error banner, receipt contents) over network captures or internal API calls.
- Network/request assertions are acceptable only when the request **is** the contract under test (e.g. a spec whose explicit purpose is verifying a backend API contract) — not as a proxy for "the flow worked".
- When a flow's implementation changes but its behaviour doesn't, existing E2E specs should keep passing without edits. If a refactor forces you to rewrite assertions, the assertions were testing mechanics.

When reviewing failed tests, always check and where possible improve tests to be outcome based.

### 4. Classify each real failure

For every failure that reproduces in isolation, decide:

- **A. Regression** — the new code broke behaviour the test correctly protects.
  → Fix the code in the **driver** worktree, re-run the affected spec to confirm green, commit, push. War Room picks it up on its own: the review-phase gate re-runs `warroom pr review`; the merge-phase gate re-enters `warroom pr review` for you (the pushed delta is app code, so it is reviewed before anything merges). You do not run either yourself.

- **B. Intended behaviour change, or a stale spec/helper/fixture** — the branch deliberately changed behaviour and the test asserts the old contract, or the harness (stub, fixture, helper) lags the product.
  → Update the test to the new contract in the worktree it lives in (driver or demo companion). Prefer asserting the **observable outcome** over internal wire calls. Commit, push, and **explain WHY on the PR** — a changed test is a changed contract, and that comment is what a reviewer reads. The gate re-runs on your pushed delta and the merge continues if it is green; the edited spec is named in the gate's summary so the reviewer sees it on the PR.

- **C. Environmental (from step 2)** — not a code or test defect. Flag on the PR; do not change code or tests. War Room re-runs the suite itself.

### 5. What War Room does with your commits

After you exit, War Room diffs each gate worktree against where it stood when you started:

- **Nothing committed** → read as an environmental call; the suite re-runs as-is.
- **Only test files** (`*.spec.*`, `*.test.*`, `tests/`, `e2e/`) **and/or static-analysis config** (knip, biome, eslint, prettier, lint-staged, editorconfig) → pushed, suite re-runs, a green result resumes the merge automatically.
- **Anything else** (app code, package manifests, lockfiles, CI) → pushed, then the merge stops and the PR goes back through `warroom pr review`. So keep app code out of the demo companion, and out of the driver unless it *is* the regression fix.
- If the repo's pre-commit hook rejects your commit on a **pre-existing** finding unrelated to your fix (e.g. knip flags a dependency you did not add), fix it in the tooling config rather than bypassing the hook — that stays resumable.
- **Commit everything you change.** Uncommitted edits are folded into a follow-up commit by War Room, but only on the PR's own branch, and a hook that rejects them leaves the gate dirty.

### 6. Report

- Summarise per failure on the PR: bucket (A/B/C), root cause, action taken — one comment, not one per spec.
- **C** that keeps failing on a settled environment → escalate: it points to real infra (single-dev-server throughput, test-account limits), not to this PR.
