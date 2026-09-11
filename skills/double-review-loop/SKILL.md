---
name: double-review-loop
description: "Run two independent read-only reviews. Use only when the user explicitly requests two reviewers, a double review, or this skill."
---

# Double Review Loop

This explicit two-reviewer gate supplements Astra Advisor only when requested.
Risk alone does not activate it. Astra Advisor selects supported models and efforts;
model-family diversity is required only if the user asks for it.

## Review the stable artifact

Establish the target, accepted requirements, constraints, known risks, validation
evidence, and whether the request is findings-only or includes fixes. Use relevant
PR history and earlier correction outcomes; verify claims located through memory.
Keep unpublished work in the conversation, without a second journal.

Give two independent reviewers the same stable artifact and necessary context,
without suggesting intended findings. Use a callable native read-only review entry
point or system review skill when available; otherwise disclose use of fresh native
read-only reviewers. A native review that internally uses several agents counts as
one lane. Do not enable fixes, comments, or other writes; the parent owns correction,
Git, and GitHub actions.

Keep the artifact stable while the pair reviews. If it changes materially, stabilize
it before replacing the affected review evidence. If the pair cannot run, report the
missing evidence; a single review does not satisfy the requested double gate.

## Triage and correct

Classify findings by evidence and scope: pertinent, not pertinent, or ambiguous.
Pertinence alone does not require a fix. Apply the accepted blocking threshold:
`fix-first` needs a demonstrated in-scope blocking defect against a user requirement
or supported contract; `ship` may include residual risks. Non-blocking P2 and P3
findings alone do not trigger correction or another review cycle.

For findings-only requests, return findings and proposed corrections. Otherwise,
batch accepted blockers by invariant and owner. Start with the failing case and
check affected sibling paths; do not weaken tests, silence errors, or add broad
fallbacks just to clear a finding. Remove abandoned task-introduced capabilities.

For repeated failures, inspect prior attempts. After two failed corrections of the
same invariant, or accumulating exceptions outside its owner, reassess reproduction,
assumptions, and ownership before another patch. Refactor only when a demonstrated
cause justifies it. Ask only for a material unresolved scope or authorization choice;
continue authorized work otherwise.

## Verify and finish

Run affected checks and return the correction delta and affected contracts to the
same reviewer or pair for targeted confirmation. Preserve unaffected evidence.
Renew full independent review only for material changes to design, authority,
ownership, or scope, not for each bounded edit. Run broader checks when required by
the repository or changed risk; do not repeat successful checks without cause.

Before commit or push, satisfy the explicitly requested gate and required validation.
For PR closure, let the GitHub skill handle publication, replies, and resolution.
Report the pair, outcome, fixes, checks, residual findings, and missing evidence.
Do not claim the double gate passed when either required review or validation is
unavailable or still blocking.
