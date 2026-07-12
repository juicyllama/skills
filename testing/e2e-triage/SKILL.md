---
name: e2e-triage
description: Triage a failing review-phase Playwright e2e run. Work through the steps to determine whether the failure is a regression, an intended behaviour change, or an environmental flake. Report your findings back to the PR and ensure the merge gate is respected.
---

Canonical, adapter-agnostic procedure for triaging a failing **review-phase Playwright e2e** run. Any agent — Claude, Codex, or a human — follows this. Per-adapter entry points import it:

- **Codex / generic agents:** referenced from [`AGENTS.md`](../AGENTS.md).
- **Claude Code:** the `warroom-e2e-triage` skill (`.claude/skills/warroom-e2e-triage/SKILL.md`) points here.
- **Any runtime:** the `warroom pr review` CLI prints a triage directive pointing here when the run fails.

## When this runs

`warroom pr review` runs the demo Playwright e2e after code review and **before** offering to merge. On failure it posts a `❌ Playwright e2e — FAILED` comment to the PR and **blocks the merge**. Work this procedure at that point. The merge stays blocked until triage is resolved.

## Non-negotiables

- **Do not merge** while any e2e test is failing. Never bypass a red gate with `--admin` or by skipping e2e.
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

**Test for it:** re-run the specific failing test(s) **in isolation, single worker, on a settled machine** (`PLAYWRIGHT_WORKERS=1 playwright test <spec> -g "<name>"`). If it passes alone, the full-run failure was environmental — do **not** change code or tests. Instead: note it on the PR, and re-run `warroom pr review` when the environment is fresh (lower `WARROOM_E2E_WORKERS`, or `repos.yaml defaults.e2e_workers`, if the box is small).

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
  → Fix the code. Re-run the affected spec to confirm green, then **re-run `warroom pr review`** to re-validate the full gate and re-open the merge prompt. This restarts the loop.

- **B. Intended behaviour change** — the branch deliberately changed behaviour and the test asserts the old contract.
  → Update the test to the new contract. Prefer asserting the **observable outcome** over internal wire calls. **Explain WHY on the PR** and **keep the merge blocked pending human sign-off** — a changed test is a changed contract and needs a human's eyes.

- **C. Environmental (from step 2)** — not a code or test defect. Flag on the PR, re-run fresh; do not change code or tests.

### 5. Report and gate

- Summarise per failure on the PR: bucket (A/B/C), root cause, action taken.
- All **A** and fixed → re-run `warroom pr review`.
- Any **B** → leave the merge blocked; hand back to the human to approve the test change.
- All **C** → re-run `warroom pr review` on a settled environment; escalate if it keeps failing (points to real infra: single-dev-server throughput, test-account limits).
