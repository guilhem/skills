---
name: simplify-by-composition
description: Simplify software designs and tooling workflows by narrowing responsibilities, reusing dedicated tools, and reducing custom maintenance. Use when a simplification or consolidation decision could replace established capabilities with bespoke code, or when deciding what a component should delegate.
---

# Simplify by composition

Optimize for the complexity the project must own across the complete solution,
accounting for the people, agents, and tools actually available. File count, tool
count, dependency count, and lines of code are evidence, not the definition of
simplicity.

## Make the decision

- Identify the requested outcome and the responsibilities actually needed. Remove
  unnecessary work before choosing how to implement it.
- Inspect existing tools, platform capabilities, and project conventions before
  proposing replacements. Prefer a suitable existing owner over custom code that
  takes over its job. Reuse is the starting point; replacing an established
  capability with bespoke code needs a concrete justification.
- Keep each component's purpose narrow and its interfaces explicit. Compose
  components through supported interfaces with as little adaptation as necessary.
- Evaluate the whole cost: custom behavior, coupling, configuration, deployment,
  upgrades, failure diagnosis, security, and the knowledge required to operate it.
  Identify costs in the actual environment rather than assuming that each tool
  adds significant burden. Distinguish using another existing command or library
  from introducing a service with its own state, deployment, and availability.
- Account for AI-assisted tool use: familiarity with established interfaces can
  make using several specialized tools cheaper than understanding and maintaining
  one bespoke implementation. Verify relevant interfaces and behavior; do not
  assume perfect tool knowledge. Tool count alone is never a reason to consolidate.
- Do not assume unaided human learning or execution limits apply unchanged to
  AI-assisted work. Unfamiliarity, novelty, and blanket rules such as "fewer
  dependencies is better" are not sufficient objections. Name the concrete
  failure mechanism or constraint and check it proportionately; label unverified
  concerns as hypotheses. Apply this standard to both reuse and custom code.
- When transferring a responsibility, name its new owner and verify that the
  required behavior is covered. Keep application policy and trust-boundary
  validation where they belong; propagate failures instead of hiding them.
- Prefer a short implementation when it fully solves a small, stable problem.
  Use it when no suitable existing capability fits, or when a concrete constraint
  makes reuse more costly. Avoid creating wrappers or process boundaries solely
  to make the design look modular. A boundary can be a function, module, library,
  or process.

## Guard against misleading simplification

- Replacing dedicated tools with one custom script can increase the number of
  responsibilities the project owns. Require a concrete benefit before doing it.
- A thin script that connects tools can be a good solution. A script that
  reimplements their scheduling, retry, parsing, packaging, or security behavior
  needs justification against the existing capabilities.
- Moving code into many files or services does not simplify a design if coupling
  and operating burden remain or grow.
- Judge dependencies by the responsibilities they remove and the obligations they
  actually introduce. Neither adding nor removing one is inherently simpler.
- Apply the same cost comparison to agent delegation: a bounded independent task
  can benefit from a specialist; a short lookup may cost less to do directly.

## Explain the result

Briefly state which responsibilities disappear, which move to an existing owner,
what custom code remains, and what integration costs or limits are introduced.
Validate the affected interfaces and failure behavior in proportion to the change.
Distinguish inspected behavior from checks actually run.

For example, using an existing scheduler to invoke a focused command can remove
custom scheduling and restart logic. Replacing a few stable library calls with a
separate service can add deployment and failure handling without removing enough
work to justify it. Choose from the actual requirements and environment.
