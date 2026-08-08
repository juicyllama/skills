---
name: completeness-check
description: Review a PR against the requirements it was opened to satisfy — the issue body, its acceptance criteria, and the triage discussion — to confirm everything asked for was delivered, nothing was quietly dropped, and nothing out of scope crept in. A feature-scope review, not a code review.
---

Canonical, adapter-agnostic procedure for a **feature scope review** of a branch submitted as a PR. Any agent — Claude, Codex, or a human — follows this. Per-adapter entry points import it:

- **Codex / generic agents:** referenced from [`AGENTS.md`](../AGENTS.md).
- **Claude Code:** the `completeness-check` skill points here.

## What this is, and what it is not

This answers exactly one question: **does this PR deliver what was asked for, and only what was asked for?**

- It is **not** a code review. Code quality, style, duplication, naming, and complexity are out of scope — the `dry` skill and CodeRabbit own those.
- It is **not** a correctness review. Whether the code *works* is the test suite's and CodeRabbit's job.
- It **is** a scope review: requirement by requirement, is it there, is it missing, or is it something nobody asked for?

The failure this catches is the one reviews miss most: code that is well written, passes every test, gets approved — and silently does four of the five things the issue asked for. Tests only prove what was written works; they say nothing about what was never written.

## When this runs

After CodeRabbit and human review have settled, and **before** the DRY/KISS pass and the Playwright gate. That ordering is deliberate: the feature should be scope-complete before anyone refactors its shape or validates it end to end. Refactoring code that is about to be deleted as out of scope — or DRY-ing a feature that is missing a third of itself — is wasted work.

## Non-negotiables

- **The requirements are the source of truth, not the code.** Read the issue body, its acceptance criteria, and the triage discussion first, and build the checklist from them *before* looking at the diff. Deriving the checklist from the PR guarantees the PR passes its own exam.
- **Judge observable outcomes, not implementation.** A requirement is met when the behaviour it describes exists and is demonstrably exercised — not when a function that looks related exists.
- **Evidence or it did not happen.** Every ticked item cites the file (and where useful the symbol or test) that satisfies it. An item you cannot evidence is not complete.
- **Report, do not implement.** This pass produces a checklist and a verdict. It does not write the missing feature, delete the extra one, or fix anything — a separate remediation step acts on the report. Keeping analysis and change apart is what makes the report trustworthy.
- **Uncertain is its own answer.** When a requirement is ambiguous, or you cannot tell whether the diff satisfies it, mark it **unclear** and say why. Never guess in either direction: a false tick hides a gap, a false gap sends someone chasing work that is already done.
- **Out of scope is not automatically wrong.** Extra work may be a necessary enabler, an obvious drive-by fix, or genuine scope creep. Decide, and justify — do not reflexively flag every unlisted change.

## Repeat checks build on the last checklist

If this review runs more than once on a PR — e.g., after each remediation fix, and again whenever the review is re-run in a later session. When it does, you will be handed **the checklist the previous pass produced**, with each item's status and evidence. Build on it; do not start over.

- **Reuse the checklist, do not re-derive it.** The items and their sources were already established from the requirements. Deriving a fresh checklist every pass is wasted work and makes the outcome wobble — an item worded one way last time, another way this time. Keep the established items; only add one if the requirements genuinely contain a requirement the checklist missed.
- **Re-confirm the done items, cheaply — never blind-trust them.** A change made to close one gap can regress or delete another requirement that was green. So re-check that each done item's evidence still holds against the *current* diff, but do not re-litigate ones that plainly still hold. If one regressed, move it back to missing/partial and say what broke.
- **Spend your effort on the items that are not yet green.** Missing, partial, and unclear items are where the real work is: re-check each against the current diff and turn it green (or explain why it still is not).
- **Only a change to the requirements resets the baseline.** If the issue itself changes — a new acceptance criterion, a requirement dropped in triage — War Room discards the old checklist and you rebuild from scratch. The requirements are still the source of truth; the carried-forward checklist is only a cache of the derivation, never a substitute for it.
- **The follow-on comment is incremental — it builds on the last one instead of repeating it.** Re-confirming the greens is silent work. Publish only the items that were not green, any carried-forward item that regressed, and any genuinely new item; collapse the rest into a single line — `Carried forward: 12 of 14 items confirmed in earlier passes re-verified and still green.` The gaps, the out-of-scope decisions, and the verdict cover that same incremental set. A reader should be able to see what *moved* since the last comment without re-reading fourteen unchanged ticks.
- **The trimming stops at the comment.** The machine-readable verdict still carries the full checklist, done items included — it is the baseline the next pass inherits.

## Process

### 1. Build the checklist from the requirements

Before opening the diff:

- Read the linked issue in full: the problem statement, the **acceptance criteria**, and any TDD/fix plan.
- Read the triage discussion and comments. Requirements are routinely refined, added, or explicitly dropped there — a comment saying "let's skip the migration for now" is as binding as the issue body, and an item dropped in triage is **not** a gap.
- Read any product documentation the issue asks for. Docs are a deliverable when requested.

Produce a flat checklist of **verifiable, outcome-shaped** items. Each is one thing that must be observably true when the PR lands. Split compound criteria ("add the endpoint and document it") into separate items — they fail independently.

Record for each item: its source (issue AC, triage comment, fix plan step) so a reader can trace it back.

#### Pipeline-enforced validation is not a requirement

Issues and fix plans routinely close with a validation step: "run `npm run go`, `npm run typecheck`, and the Playwright suite, and confirm they pass." Do **not** turn these into checklist items, gaps, or escalations — not even as a "needs a human at merge time" item. The delivery pipeline already enforces them mechanically: the husky pre-commit hook runs the `npm run go` gate on every commit, and War Room's merge gate runs typecheck and the Playwright e2e suites before any merge. The commands cannot be skipped, so an item asking someone to run them and confirm adds nothing — it only stalls the review on a human decision the machine has already made.

The exclusion covers *running commands the pipeline already runs* (build, lint, typecheck, unit tests, the e2e suites). It does not cover:

- **Writing** a test or spec the requirements ask for. "Add an e2e spec for partner referral links" is a deliverable — whether the spec *exists* is your checklist item; whether the suite *passes* is the pipeline's job.
- Validation the pipeline does **not** perform — a manual smoke test on a live environment, a check in another system, release-time verification of infra or ops. Those stay on the checklist and route per the rules below.

### 2. Walk the PR's changes against the checklist

- Get the branch's full diff against the PR's base (`git diff <base>...HEAD`, or the PR's changed files).
- For each checklist item, hunt the diff for what satisfies it — implementation, and where the requirement implies it, the test that exercises it and the documentation that describes it.
- Mark each item:
  - **done** — with the file/symbol/test that evidences it.
  - **missing** — nothing in the diff satisfies it.
  - **partial** — some of it landed; say precisely what is absent.
  - **unclear** — you cannot tell; say what would settle it.

Work item-by-item over the checklist, not file-by-file over the diff. Reading the diff and asking "what does this do?" reconstructs the author's intent and will quietly re-derive their scope — the exact bias this pass exists to defeat.

### 3. Compile the gaps

Collect every **missing**, **partial**, and **unclear** item into a breakdown. For each, state:

- The requirement, and where it came from.
- What is actually there now (nothing / what partially exists).
- What would satisfy it — concretely enough for someone to act on without re-reading the whole issue.

If there are no gaps, say so plainly. A clean result is a real result.

#### Route each gap to whoever can close it

This decides what happens next, so it matters as much as finding the gap: a gap routed to an agent goes straight to a remediation session, and one routed to a human blocks the merge until they resolve or accept it. Route it wrong in either direction and the review stalls — an agent is handed work it cannot do, or a person is asked to decide something the branch could have fixed.

- **Agent-closable.** It can be closed by editing this repository from this checkout. The default for ordinary undelivered work.
- **Needs a human.** It cannot be closed or verified from here: live infrastructure or ops (dashboards, alerts, retention, live error tracking), work owned by another repository, or manual and release-time verification.

One case sits between the two and is the one most often routed wrong: **the requirement is built, but nothing in a production execution path calls it** — an event, webhook, job, scheduler, or dispatcher. It looks like ordinary in-repo work, so it gets handed to an agent, which then declines to design a production integration nobody specified. Ask who chose that integration point:

- **The requirements name it** — an AC, the fix plan, or a triage comment says this must happen on that event or path. It is approved work: agent-closable, and say exactly where to hook it in.
- **You inferred it** — the issue asked for the capability and you concluded it is inert unless wired. That is a scope judgement, not an undelivered requirement: it needs a human, and name the candidate call sites and what each would change at runtime so they can settle it in one read.

Either way, if hooking it in means editing an execution path this PR does not already touch, say so in the item. Choosing where a lifecycle event calls new code carries production blast radius, and an agent must not decide that alone.

### 4. Find what landed that nobody asked for

Walk the diff again for substantive changes that map to **no** checklist item. Ignore incidental mechanics (imports, formatting, lockfiles, moved code). For each, decide:

- **Keep — enabler.** The requirement could not be met without it. It is in scope in practice.
- **Keep — justified.** Small, sound, clearly beneficial, low risk (a typo fix, a missing null guard next to touched code). Note it so a human can disagree.
- **Remove — scope creep.** Unrelated work that belongs in its own issue: it inflates the diff, ships unreviewed risk, and hides behind the feature's approval. Say what to remove and why.
- **Escalate — undiscussed behaviour change.** It changes behaviour nobody signed off on. Do not decide alone; flag it for a human explicitly.

Bias toward **keep** for anything small and defensible, and toward **remove/escalate** for anything that carries its own risk or would surprise the issue's author.

### 5. Publish the verdict

Post one PR comment containing:

- **The checklist** — every item with its status and, for done items, the evidence. On a repeat pass, only the incremental items (see above).
- **The gaps** — the step 3 breakdown, or an explicit "no gaps found".
- **The extras** — each unlisted change with its decision and reasoning.
- **The verdict** — `complete` (every item done, no removals needed) or `incomplete` (anything missing/partial/unclear, or anything to remove/escalate).

Write each checklist item in one shape, every pass, so a reader can scan a review's comments side by side:

```
- ✅ **Done** — <the requirement, one line>
  Source: <issue AC / triage comment / fix-plan step>
  Evidence: <file, symbol, or test that satisfies it>
```

The status vocabulary is fixed: `✅ **Done**`, `⚠️ **Partial**`, `❌ **Missing**`, `❓ **Unclear**`. Never publish a bracketed `[DONE]`-style status — War Room hands a repeat pass its baseline in this same format, and copying a different one through breaks the sequence.

Write it for the issue's author: they must be able to tell, without reading the diff, whether they are getting what they asked for.

Then write the same verdict to the structured output path War Room provides, if it provided one. That file is what the remediation step consumes — the PR comment is for humans, the file is for the machine. If both exist they must agree.

Include the **full checklist** in that machine-readable file, not just the gaps: every item with its status (`done`/`missing`/`partial`/`unclear`), its source, and its evidence. War Room persists it as the baseline the next pass builds on, so the done items must be there too — a checklist that only lists the gaps forces the next pass to re-derive everything.

### 6. Hand off

Stop here. Do not implement gaps, remove extras, or refactor — a separate remediation session acts on this report, and a human decides anything you escalated. A scope review never merges, and never quietly fixes what it found.
