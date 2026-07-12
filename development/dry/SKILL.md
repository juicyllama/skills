---
name: dry
description: Refactor a developed feature branch (submitted as a PR) using DRY and KISS principles. Find duplicated code and over-complicated constructs introduced by the branch, extract simple abstractions, and simplify — without changing the functionality or observable outcome of the code.
---

Canonical, adapter-agnostic procedure for a **behaviour-preserving refactor pass** over a feature branch that has been submitted as a PR. Any agent — Claude, Codex, or a human — follows this. Per-adapter entry points import it:

- **Codex / generic agents:** referenced from [`AGENTS.md`](../AGENTS.md).
- **Claude Code:** the `dry` skill points here.

## When this runs

After a feature is **functionally complete** on its feature branch and a PR exists (or is about to be opened), and before final review/merge. The feature works; this pass makes the code it added easier to understand and develop by removing repetition (**DRY** — Don't Repeat Yourself) and unnecessary complexity (**KISS** — Keep It Simple, Stupid).

## Non-negotiables

- **Never change functionality or outcomes.** Every refactor must be observationally equivalent: same inputs → same outputs, same side effects, same errors, same user-visible behaviour. If you cannot argue equivalence, do not make the change.
- **Do not touch behaviour to "improve" it.** Bugs, missing validation, or design flaws you notice are **reported, not fixed** — flag them on the PR for a human; fixing them here muddies the refactor diff.
- **Stay in scope.** Refactor only code the branch added or modified (plus the minimal shared code needed to host an extraction). Do not launch a repo-wide cleanup.
- **Do not weaken or rewrite tests.** Tests may only change mechanically when a refactor moves/renames a symbol they reference. Assertions stay identical. The full test suite must be green before and after.
- **Abstractions must be simple.** Extract only when duplication is real and the abstraction is more understandable than the repetition. No speculative generality, no config flags "for later", no deep inheritance. When DRY and KISS conflict, KISS wins — a small amount of honest duplication beats a clever abstraction.
- **Match the surrounding code's conventions** — naming, structure, idiom, comment density.

## Process

### 1. Establish the baseline

- Check out the feature branch and identify its base (`git merge-base` against the PR's target).
- `git diff <base>...HEAD` to get the full set of files/hunks the branch introduced. This diff **is** your scope.
- Run the project's test suite, lint, and typecheck. Record the result. If anything is already red, stop and report — you need a green baseline to prove behaviour is preserved.

### 2. Hunt for DRY violations

Scan the branch's changes (and how they interact with existing code) for repetition:

- **Copy-paste blocks** — identical or near-identical logic in two or more places (same statements with different variable names count).
- **Parallel structures** — functions/components/handlers that differ only in a value, type, or one branch; candidates for a parameter, a small helper, or a lookup table.
- **Repeated literals** — magic numbers, strings, selectors, URLs, or config repeated across the diff; candidates for a named constant.
- **Duplication against existing code** — the branch re-implements a helper/utility that already exists in the codebase. Prefer using the existing one over extracting a new one.
- **Repeated boilerplate** — the same setup/teardown, error-handling wrapper, or mapping logic around every call site.

For each candidate, note the locations and the shape of the proposed extraction.

### 3. Hunt for KISS violations

Scan the same scope for complexity that doesn't pay for itself:

- **Needless indirection** — layers, wrappers, or interfaces with a single caller/implementation that add no meaning.
- **Over-general code** — parameters, options, or generics nothing uses; code written for hypothetical future needs.
- **Convoluted control flow** — deep nesting flattenable with early returns/guard clauses; boolean gymnastics simplifiable with named intermediates; clever one-liners clearer as two plain statements.
- **Dead weight** — unused variables, unreachable branches, commented-out code, redundant conditions the branch introduced.
- **Reinvented wheels** — hand-rolled logic the language, stdlib, or an already-used dependency provides directly.

The test for every candidate: *would the next developer understand and modify this faster after the change?* If not, skip it.

### 4. Judge each candidate before touching code

For every candidate from steps 2–3, decide:

- **A. Refactor** — clear duplication or complexity, the fix is simple, and equivalence is easy to argue. Proceed.
- **B. Leave, with a note** — real but the abstraction would be speculative, the duplication is only two instances of trivial code, or the fix bleeds outside the branch's scope. Mention it on the PR; do not change it.
- **C. Behaviour issue, not a refactor** — you found a bug or inconsistency. Report it on the PR; do not fix it here.

Rules of thumb: duplication generally earns an abstraction at the **third** occurrence unless the block is substantial; an abstraction that needs its own explanation to justify itself fails KISS.

### 5. Refactor in small, verifiable steps

- Apply **one refactor at a time**, as its own commit with a message saying what was deduplicated/simplified and why it is behaviour-preserving.
- After **each** step, re-run the tests (at minimum those covering the touched code; the full suite before pushing) plus lint/typecheck. Any failure → revert or fix the refactor, never the test.
- Keep extractions close to their call sites (same module/package) unless they are genuinely shared.
- Preserve public API signatures, exported names, serialized formats, logged messages, and error types/messages unless every consumer is inside the branch's scope — these are observable behaviour.

### 6. Verify equivalence and report

- Run the **full** test suite, lint, and typecheck; confirm the same green result as the step 1 baseline.
- Re-read the final `git diff` of your refactor commits: it should contain **only** structural changes — no logic additions, no behaviour tweaks, no test-assertion changes.
- Summarise on the PR:
  - Each refactor applied: what was duplicated/complex, what it became, why the outcome is unchanged.
  - Each **B** (left, with a note) and **C** (behaviour issue found) item, so humans can decide.
  - Confirmation that the suite is green before and after.
- Hand back to the normal review/merge flow. A refactor pass never merges its own changes without review.
