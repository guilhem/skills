---
name: readme-craft
description: "Create, improve, or review project READMEs around purpose, useful capabilities, a first result, and an expressive visual identity with meaningful diagrams and badges. Use for README content and presentation; keep targeted edits proportional and review-only requests read-only."
---

# README Craft

Make the repository's front page help its readers decide whether the project fits
their need, achieve a first useful result, and find the depth needed to adopt it.
Explain what readers can do and why the project exists through clear writing,
real proof, and a graphical reading path. Adapt the reading order to the project
rather than filling a universal template.

This skill is self-contained. Use available repository tooling for checks and
previews; another writing skill, image generator, plugin, or installed renderer
is not required.

## Establish the task and the reader

Identify whether the request is creation, a refresh, a targeted addition, or a
review. Preserve the requested scope: a small correction does not authorize a
redesign; a review returns findings without edits. README work does not itself
authorize publishing, deploying, or changing global agent instructions.

Inspect the existing README, relevant source and metadata, examples, documentation,
installation paths, tests, and available assets. Use the current target revision
as evidence. Identify the actual readers and their tasks, including users,
operators, integrators, and contributors only where relevant. Ask only for missing
information that materially changes the result and cannot be established from
the request or repository; skip routine opening and closing interviews.

Before composing a substantial rewrite, establish:

- The reader's situation, the problem, and the outcome the project enables.
- The strongest available proof: an example, real output, screenshot, or diagram.
- The shortest supported route to a first useful result and recognizable success.
- Requirements, concrete usage constraints, or operational effects that affect use.

Use these answers to choose what earns space. A list of technologies is supporting
context; explain capabilities through what readers can accomplish with them.

## Compose the reading path

Make the opening explain the project's purpose, who it serves, and what readers
can accomplish in plain language. Put convincing proof close to that explanation.
A screenshot is useful for an interface; an input/output example may serve a
library better.
Keep essential meaning in text even when images fail.

Give the reader prerequisites, a complete supported example, and the expected
outcome before implementation detail. Explain enough configuration, integrations,
failure recovery, and limits to complete the real task. Keep decision-critical
information beside the affected step. Offer distinct paths for other audiences
when their tasks differ; link to deeper documentation from the relevant context.

Let readers assess maturity themselves. GitHub Releases owns release information;
give access through a link or dynamic badge instead of copying version numbers,
release notes, summaries, or release-stage commentary into the README. Do not
add maturity verdicts or generic readiness warnings.

For a substantial rewrite or questions about depth and audience paths, read
[content design](references/content-design.md). Retain useful existing detail;
move exhaustive reference only when its new location helps the reader and is
within scope. Do not impose a word count or create empty documentation links.

For calibration, read only the relevant annotated example:

- [Application](examples/application.md): a browser task and visible result.
- [Library](examples/library.md): a concrete API call and integration decisions.
- [Embedded system](examples/embedded-system.md): host simulation, installation,
  and the boundary between software checks and physical behavior.

These examples are fictional illustrations, not installable projects or required
outlines. Derive every published command, claim, and asset from the actual project.

## Design the presentation

For creation or a substantial refresh, actively design a graphical reading path.
Use Mermaid to explain a useful workflow or architecture, badges for relevant
release versions and checks, and screenshots or output to show capabilities
when they help readers understand the project quickly. Select visuals by the
information they convey; do not add them just to fill a template.

Use descriptive headings, short connected paragraphs, and tables for comparisons.
Keep the page easy to scan while retaining the detail needed for use. For creation,
a substantial refresh, or scoped visual work, read [visual design](references/visual-design.md).
Choose a coherent direction from the project's own interface, artifacts, domain,
or existing identity. Each visual should demonstrate, explain, compare, or orient.
Prefer real screenshots and outputs; label mockups and illustrative diagrams.
Keep commands, essential explanations, and navigation in readable Markdown.

Match visual effort to the task. Preserve strong existing assets. Create an asset
only when it has a concrete communication purpose; generated art must never pose
as a product screenshot, hardware result, benchmark, or other evidence.

## Verify the result

Use [verification](references/verification.md) for the final pass, or when reviewing
an existing README. Check claims and commands against the current project, follow
links, and inspect the rendered changes. Distinguish source inspection, local
execution, CI, deployment observation, and hardware validation. Report unavailable
checks without inventing successful results.

Before finishing, confirm that each relevant reader can understand the value, find
their first action, recognize success, and discover the necessary detail without
reading internal design documents. Report the changed files, meaningful checks,
and remaining limits. For review-only work, return concrete findings and their
locations instead of rewriting the README.
