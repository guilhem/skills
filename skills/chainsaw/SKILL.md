---
name: chainsaw
description: "Write, review, debug, and integrate Kubernetes tests with Kyverno Chainsaw. Use for Chainsaw Test manifests, assertions, fixtures, cleanup, multi-cluster execution, and local or CI test integration. Does not cover unrelated tools named Chainsaw or general cluster provisioning."
---

# Kyverno Chainsaw

Produce tests that fail for the intended reason, observe the controller's actual
behavior, and leave owned resources behind only when explicitly requested.
Apply strong defaults; an exception needs a reason, a compensating control, and
validation of the affected behavior.

## Establish the test contract

Inspect the project's test targets, Chainsaw version, configuration, manifests,
and CI before editing. Identify the behavior being tested, its positive and
negative outcomes, asynchronous boundaries, target clusters, and resource owners.
Ask only for consequential information the project does not establish.

Use the project's pinned binary and matching schemas. Reference examples target
Chainsaw **v0.2.15**: `Test` uses `chainsaw.kyverno.io/v1alpha1`, and the modern
`Configuration` uses `chainsaw.kyverno.io/v1alpha2`. Check the installed version
and CLI help before using a version-dependent feature. Flag a mismatch; do not
silently upgrade the project or add compatibility machinery.

## Prevent failures by default

- Prefer explicit `chainsaw-test.yaml` Tests and native resource operations.
  Put dependent phases in one Test's ordered steps or an explicit existing
  orchestrator; separate Test filenames do not establish dependencies.
- Assert the controller's observable result, not just the fields the test applied.
  Use bounded retrying assertions for convergence. A fixed sleep is not evidence
  of readiness, and increasing timeouts does not fix expression errors.
- Make missing intermediate fields a normal mismatch. Preserve types; a fallback
  must not turn missing or malformed data into a passing assertion.
- Select the intended resources. A resource assertion can pass on **one** matching
  candidate. Collection contracts need membership and, when required, cardinality
  checks before checking each item's properties.
- Verify the intended failure in negative tests. Prefer `create` for creation
  rejection; check a stable rejection identity/reason as well as an error.
  `error` means no selected object matches an assertion, not API rejection or
  absence throughout a time interval.
- A positive command/script `check` must include execution success, such as
  `($error == null): true`, as well as output checks. A custom check replaces the
  default process-error decision. Execution timeout is not automatic polling.
- Default to ephemeral test namespaces. Namespace isolation does not cover
  cluster-scoped objects or global policy effects. Isolate actual conflicts or
  serialize the affected tests; do not disable concurrency indiscriminately.
- Track ownership and teardown dependencies. Native creation is tracked;
  updating a pre-existing object with `apply` does not make it cleanup-owned.
  Collect diagnostics before teardown and keep controllers available for finalizers.
- Respect binding scope and evaluation order. Keep cluster selection deterministic,
  establish namespaces on secondary clusters, and use supported template overrides.
- Preserve process exit status and cleanup failures in CI. Keep useful console
  diagnostics alongside structured reports; pin the tested tool versions.

## Read only the relevant detail

| Task or uncertainty | Reference |
| --- | --- |
| Matching, nulls, collections, negative tests, process checks | [Assertions](references/assertions.md) |
| Ordered steps, concurrency, ownership, diagnostics, teardown | [Lifecycle and isolation](references/lifecycle-and-isolation.md) |
| Bindings, cluster targeting, local tools, CI configuration | [Integrations](references/integrations.md) |
| Why a rule exists, affected versions, historical fixes, unresolved reports | [Evidence](references/evidence.md) |

## Author

Translate the contract into the smallest explicit scenario with named steps and
targeted assertions. Reuse existing fixtures and targets without inheriting their
isolation or cleanup mistakes. Add a counterexample for non-trivial assertions:
the wrong result, a missing member, or an unrelated error must fail.
Keep asynchronous reads inside retrying operations; a captured output is a snapshot.

## Review

Trace selection, mutation, observation, and cleanup for each scenario. Report
concrete trigger → failure → correction, with the relevant source or reproduction.
Prioritize false passes, destructive ownership mistakes, races, and hidden process
failures. Keep historical corrected bugs and preferences distinct from current
defects. Preserve justified exceptions instead of performing unrelated refactors.

## Diagnose

Reproduce with the same version, configuration, cluster selection, and fixture
state. Distinguish lookup/API errors, assertion mismatches, expression errors,
process failures, controller convergence, and cleanup failures before changing
timeouts or concurrency. Narrow the failing test; inspect events, resource state,
and controller diagnostics before resources disappear. Read the evidence reference
when a symptom resembles a reported engine issue; verify applicability first.

## Validate and report

Lint complete Test and Configuration documents with `chainsaw lint test -f FILE`
and `chainsaw lint configuration -f FILE`. Lint validates schema, not controller
behavior, expression semantics, or fixture availability.

Run the affected tests through the existing project target. Use controlled
clusterless fixtures for process checks; `--no-cluster` still executes scripts and
commands. Exercise cluster-dependent behavior on the intended test environment.
For isolation-sensitive changes, repeat affected tests with the project's normal
parallelism and verify cleanup. Distinguish lint, clusterless execution, cluster
integration, and actual CI evidence; state checks that could not run.

For Kind, controller installation, or general CI setup, recommend an appropriate
available skill after checking its publisher and scope, or link official docs.
Keep this skill focused on Chainsaw integration; do not install skills or reproduce
an environment-provisioning walkthrough as part of ordinary test work.
