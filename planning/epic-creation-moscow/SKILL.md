---
name: epic-creation-moscow
description: Triage the business requirements for a task or project into a MoSCoW-prioritised epic — Must/Should/Could become GitHub issues, Won't have is filed and closed as not planned. Use when scoping a new feature set, breaking down a project brief, or planning an epic.
---

# MoSCoW Epic Creation

Turn a business brief into a prioritised, buildable feature set. This is **business triage, not technical triage** — you are deciding *what earns a place in this release* and *why*, not how to implement it. Technical planning happens later.

## Non-negotiables

- **Prioritise from business value, not implementation cost.** "Hard to build" is not a reason to demote something to Could. Cost belongs in the rationale, never in the tier decision.
- **Must have means the release genuinely fails without it.** If the epic still ships and delivers value without an item, it is not a Must.
- **Every item carries an explicit rationale naming the business driver** — the user need, revenue impact, compliance obligation, or risk it addresses.
- **Won't have items are filed, then closed as not planned — never left open, never deleted.** They are the most valuable output of the pass: the scope you consciously bought back, kept as real issues so the decision is on record, searchable and reopenable when it is time. They stay off the Campaign Map — the board tracks this release. An open Won't have is how deferred scope gets built by accident: anything that lists open issues will pick it up.
- **Write issue bodies as specifications, not diffs.** No file paths, no function names, no line numbers.

## The MoSCoW tiers

| Tier | Test to apply | Becomes |
|---|---|---|
| **Must have** | The release is a failure without it. No workaround exists. Legal, safety, or contractual obligation. | Feature issue, priority **High** |
| **Should have** | Painful to omit, but a workaround exists. Important, not vital. Would be a Must in a later release. | Feature issue, priority **Medium** |
| **Could have** | Desirable. Small impact if left out. The first thing dropped when time runs short. | Feature issue, priority **Low** |
| **Won't have** | Explicitly out of scope *for this epic*. Agreed, deferred, not built now. | Feature issue, **no priority** — filed in its repo and closed as `not_planned` the moment it is filed; kept OFF the Campaign Map, not triaged, not linked to the epic. |

Every tier becomes a real GitHub issue, so **every item needs a repo and a body**, won't-have included. The difference is what happens next: Must/Should/Could get a priority, go onto the Campaign Map, and go through technical triage; Won't have is closed as `not_planned` as it is filed, so it sits in its repo's record as future work nobody picks up by accident, until someone reopens it.

### Filing a Won't have

File it, then close it as not planned in the same pass: create the issue, then straight after, on the number that call returned, close it with reason `not_planned`. If the close fails after the create landed, retry it; if it still fails, stop and report the open issue number rather than leaving it open. No marker label is needed, and none should be added: the closed state is the marking, and a labelled issue that is still open is the one a picker lists and takes by mistake. Reopening a closed Won't have is how it later earns a place in a release.

### Sizing the tiers

A healthy epic is roughly **60% Must / 20% Should / 20% Could** by effort. If nearly everything landed in Must, you have not triaged — go back and apply the "does the release genuinely fail?" test honestly to each one. An epic with no Won't have items is a warning sign: you have not identified any scope to buy back.

An epic made entirely of Won't have items is rejected — if nothing is in scope, there is no epic.

## Process

### 1. Understand the brief

Read the brief you were given. Where it is ambiguous, resolve it from the codebase and the repo descriptions available to you rather than asking — this runs mostly hands-off.

Establish before triaging:

- **The outcome** the epic delivers, phrased as a user- or business-visible change.
- **Who it is for** — which user, customer, or internal role.
- **What "done" means** for the epic as a whole.
- **The constraints** — deadline, compliance, dependencies on other work.

If the brief is too thin to answer *any* of these, ask **one** consolidated question, then proceed.

### 2. Decompose into candidate requirements

Break the outcome into discrete, independently valuable requirements. Each one should be:

- **Vertically sliced** — delivers observable value on its own, not "the database layer".
- **Independently shippable** — can land without its siblings wherever possible.
- **Owned by exactly one repo** among the mapped repos available to you.

For smaller tasks, you may need to combine them into a single requirement. For larger ones, you may need to break them down further. The goal is a set of candidate requirements that are **roughly equal in size and value**.

Items should be split out by repo/application, so that each repo has a clear set of issues to work on. If a requirement spans multiple repos, it must be split into separate items.

### 3. Assign a MoSCoW tier to each

Apply the test in the table above to each candidate, honestly and one at a time.

For each item, write the rationale as: **the business driver**, then **the consequence of omitting it**. That second half is what makes the tier defensible — and it is what a reader will use to challenge you.

Actively look for Won't have items. Common sources: gold-plating on a Must, a second delivery channel, an admin UI for something usable via API, edge cases affecting a negligible share of users, anything that is really a *separate* epic.

### 4. Order within each tier

Within a tier, order by dependency first (things others need come earlier), then by business value. The order you emit is the order the issues are created and later triaged, so it should read as a sensible build sequence.

### 5. Write each issue

Every item becomes a Feature issue body:

```markdown
## Business outcome

What changes for the user or the business when this lands. One or two sentences.

## Why this is a <Must|Should|Could> have

The business driver, and what happens if this is omitted.

## Scope

- What is included
- What is explicitly not included

## Acceptance criteria

- [ ] Observable, testable criterion
- [ ] Observable, testable criterion

## Open questions

- Anything technical triage needs to resolve
```

Keep bodies free of implementation detail — the issue must survive a refactor. Technical design outside the scope of this skill.

**Won't-have bodies are records of a closed decision**, so write them for whoever reopens one months from now with no memory of this epic. Same template, with two changes: the "Why" section states why it was deferred *from this epic* and what would have to change for it to earn a place in a future one; acceptance criteria can be thinner, since this issue will be triaged properly if it is reopened. Name the epic it was cut from.

### 6. Write the epic

The epic is the parent tracking issue. Its body states the outcome, the scope boundary, and how success is measured. War Room appends the child issue table itself — **do not write the table yourself.**

**When the epic already exists.** If your prompt says the epic is an existing issue being promoted, skip this step entirely and emit no `epic` key. That issue's title and body are a human's writing and are kept verbatim — do not restate, rewrite, or "improve" them. Your job is only the decomposition. Treat the source issue and its discussion as the brief: requirements agreed in the comments count as much as the ones in the body.

## Output contract

Write **only** a JSON object to the draft path given in your prompt. No Markdown fences, no prose before or after.

```json
{
  "epic": {
    "title": "Short outcome-focused epic title",
    "repo": "owner/repo",
    "body": "Markdown: ## Outcome / ## Scope boundary / ## Success measures / ## Constraints"
  },
  "items": [
    {
      "moscow": "must",
      "title": "Short outcome-focused issue title",
      "repo": "owner/repo",
      "body": "Markdown body following the issue template above.",
      "rationale": "Business driver, and the consequence of omitting it."
    },
    {
      "moscow": "wont",
      "title": "Short title of the deferred scope",
      "repo": "owner/repo",
      "body": "Markdown body for the closed record: what it is, why it was cut from this epic, what would earn it a place later.",
      "rationale": "Why this is out of scope for this epic, and when it should be revisited."
    }
  ]
}
```

Rules for the JSON:

- `moscow` is exactly one of `must`, `should`, `could`, `wont`.
- `repo` and `body` are **required on every item**, won't-have included — every tier becomes an issue. Never `null`; an item missing either is dropped.
- `repo` must be one of the mapped repos listed in your prompt — never invent one. A won't-have still needs the repo that would own the work.
- `rationale` is required on every item.
- Emit items grouped and ordered: all `must` first, then `should`, then `could`, then `wont`.
- The epic `repo` is supplied to you in the prompt. Use it verbatim.
- **Omit the `epic` key entirely when promoting an existing issue.** Anything you put there is discarded, and the schema does not require it in that mode.
- At least one item must be `must`, `should`, or `could`.

## What happens next

War Room takes your JSON and:

1. Creates **every** item as a **Feature** issue in its repo. Must/Should/Could are labelled with their tier (`warroom-must` / `warroom-should` / `warroom-could`) so the MoSCoW decision is readable on GitHub itself, and go onto the Campaign Map board at `needs-triage` with priority High/Medium/Low. Won't have gets neither a label nor a place on the board: it is created and then closed as `not_planned` in the same pass, so months of deferred scope cannot crowd out the work of this release.
2. Creates (or promotes) the epic as a **Planning** issue, links the Must/Should/Could children to it as sub-issues, appends the child table to its body, and adds it to the Roadmap board. Won't-have issues are deliberately **not** linked — they are a closed record, not part of delivering this epic.
3. Runs `warroom issue triage` over each Must → Should → Could child. Won't-have issues are skipped — they stay closed until someone reopens one to put it into a release.

Won't have items appear in the epic table with their issue link and no priority. That record is the point — it is the scope decision, written down, and the closed issue is where it gets reopened.