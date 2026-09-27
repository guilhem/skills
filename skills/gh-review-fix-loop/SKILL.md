---
name: gh-review-fix-loop
description: "Close GitHub PR feedback through triage, fixes, validation, and thread resolution. Use for requested review closure, not simple inspection or selected fixes."
---

# GH Review Fix Loop

Complete the requested PR review-closure workflow: inspect feedback, correct
in-scope blockers, validate, publish, reply, react, and resolve eligible threads.
Invoking this skill requests that workflow. Approval and merge require an explicit
request. Before the first write, identify the PR and intended publication and
closure actions; do not ask again for already authorized steps.

Astra Advisor owns native delegation and local review. This skill owns GitHub triage
and closure, without adding a second review gate. Use `double-review-loop` only
when explicitly requested. For inspection or selected fixes without closure, use
GitHub tools directly.

## Establish scope and evidence

Resolve the PR from an explicit URL or number, otherwise from the current branch
and upstream. Ask only if multiple plausible targets remain. Retrieve comments and
review threads with pagination, including resolved or outdated discussions when
relevant to earlier corrections. Use the current diff, linked commits, checks, and
product constraints as evidence. Memory may locate history but does not prove it.

If a CI job does not start or has no steps, inspect its check-run annotations
before looking for logs or waiting. Use that evidence to distinguish code failures
from external blockers such as quota, billing, or permissions. Do not rerun
blindly or change code to address an external blocker; report it without
claiming CI passed.

Classify findings before editing:

- `pertinent`: a demonstrated defect or violated contract within the requested scope.
- `not pertinent`: contradicted, already handled, outdated, duplicated, out of scope,
  or unsupported subjective style advice.
- `ambiguous`: plausible but not established; retain the uncertainty and leave open.

Relevance alone does not make a finding blocking. Apply the user's and `AGENTS.md`
thresholds: `fix-first` needs a demonstrated in-scope blocking defect; `ship` may
include residual risks. Non-blocking P2 and P3 findings alone do not start another
correction or review cycle. Missing tests or checklist entries alone are evidence
gaps, not defects. Do not implement machinery solely to satisfy an invented promise.

## Correct the cause

Group blockers by behavioral invariant and owner; combine overlapping fixes.
For recurring failures, inspect the relevant earlier attempts and affected sibling
paths. After two implemented corrections fail the same invariant, or exceptions
accumulate outside its owner, reassess reproduction, assumptions, and ownership
before another patch. A repeated comment is not another failed correction.

Choose a local fix, a scoped refactor, or further diagnosis from the evidence.
Refactoring needs a demonstrated cause, the complexity it removes, and why a local
fix is insufficient. Ask for direction only when a material product, authority,
ownership, migration, or rollout decision exceeds the authorized scope. Continue
independent permitted work when history or access is unavailable.

Work directly unless independent fixes justify delegation. Let Astra Advisor route
those assignments with exact ownership, constraints, prior outcomes, and acceptance
evidence. Parallel writers use isolated worktrees; sequence dependent or overlapping
changes, and work sequentially if isolation is unavailable. The coordinator owns
integration, Git writes, and GitHub actions.

Inspect the integrated diff and run affected checks. Apply Astra Advisor's review
policy to the stable correction artifact: bounded fixes need targeted confirmation,
not a new full review solely because the original PR was substantial. Preserve
unaffected evidence. Remove capabilities introduced and abandoned within this task.

## Publish and close

Publish one integrated, validated, reviewed correction per round, never one push per
thread. Stage only intended files and verify the remote head before claiming the fix
is published or resolving feedback. Follow requested checks and reviews; avoid
duplicate requests on an unchanged head.

Check existing replies, reactions, and resolution state before writing:

- Fixed blocker: reply with the change and published commit/check evidence, add `+1`
  to the original comment, then resolve. For a changed diagnosis, explain the prior
  attempt, remaining cause, and chosen correction. Several threads may link one
  shared explanation.
- Not pertinent: give concise evidence, add `-1`, and resolve only when demonstrably
  non-actionable.
- Accepted residual or deferred finding: report the remaining risk; resolve only
  when the user explicitly accepts closure or the thread is demonstrably satisfied.
- Ambiguous or still requiring a fix: leave open; reply with the missing evidence or
  material decision when useful. Do not react or resolve without user direction.

Keep the PR description about final behavior and validation; preserve earlier design
decisions in linked discussions, without a second task journal.
Prefer supported GitHub connector operations; use `gh api graphql` for thread-level
actions when needed. Report authentication, permission, or rate-limit blockers and
continue unaffected work; request user action only when needed to proceed.

Finish with the PR, fixes, validation, review outcome, and any remaining threads or
risks with reasons. Do not claim closure or a passed gate without the required evidence.
