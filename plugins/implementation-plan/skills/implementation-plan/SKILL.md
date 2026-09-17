---
name: implementation-plan
description: Write and revise repository-grounded implementation plans for an agent without conversation history, including in Plan mode. Use for implementation planning, not a simple task list or execution of an already accepted plan.
---

# Implementation Plan

Produce a plan an agent with the repository, but no conversation history, can
execute. Write in the user's requested language. The plan is the deliverable;
using this skill does not authorize implementing it.

Scale detail to the change. Use a structure that helps the reader; a small change
does not need every heading of a large project. Resolve consequential decisions
before handing off; leave mechanical choices to the executor. Ask for missing
information only when it materially changes the outcome and cannot be established
from the repository or request. Label remaining assumptions and their impact.

## Ground the plan

- State the problem, who benefits, the observable change, and success criteria.
  Preserve that objective through revisions unless the request actually changes.
- Inspect relevant entry points, current behavior, contracts, and applicable
  repository guidance. Distinguish verified facts from assumptions. Include
  selected decisions and rationale where it helps the executor choose correctly.
- Organize changes by behavior or responsibility in a useful implementation
  order. Describe interfaces, data flow, and dependencies where omission would
  leave an important decision unresolved. Verify paths and symbols used as
  locators; clearly label proposed new files and symbols.
- For non-obvious work, point to a relevant repository example or include a short
  code or pseudocode excerpt. Mark illustrative code and invented names explicitly.
  When citing a guide, include its path and the concrete rule to apply, not just
  an instruction to read it.
- Explain only pitfalls supported by this change and observed evidence. Connect
  each situation to its consequence and a concrete prevention. Avoid generic risk
  lists and speculative mechanisms.
- Define proof of success: scenarios, commands, and expected results covering the
  requested behavior and identified difficulties. Check commands against the
  repository's actual tooling. Separate checks already run from checks the
  executor must run; never present proposed validation as successful execution.

Consult [the illustrative example](references/example-plan.md) when calibrating
how much implementation guidance, a concrete pitfall, and acceptance evidence to
include. Its fictional paths and design are examples, not facts about the target
repository or a mandatory outline.

## Revise the whole artifact

Return a complete, coherent replacement plan, not an amendment or a list of edits.
Integrate feedback in the affected sections according to its actual impact while
preserving still-useful detail from earlier versions.

Remove abandoned options and their now-unnecessary consequences throughout the
plan: examples, implementation steps, tests, and documentation. Do not
mechanically retain rejected options in an “out of scope” section. Preserve an
exclusion only when it remains a meaningful boundary of the current request.
Remove references to conversation history and phrases such as “as discussed.”

After any change, reread the entire plan for consistency between the objective,
implementation, examples, pitfalls, and validation. A minor wording correction
belongs in the relevant detail; it must not become the main priority or erase
the original feature's implementation guidance.

## Check the handoff

Read the final plan as an agent encountering the subject for the first time.
Can it understand the intended result, locate the necessary repository elements,
and start implementing without asking for information already established?
Resolve concrete gaps, stale alternatives, unsupported claims, and contradictory
steps. Do not add headings or process solely to make the plan look exhaustive.
