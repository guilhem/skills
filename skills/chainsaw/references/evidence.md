# Evidence ledger

Researched **2026-10-13**. The source baseline is release
**v0.2.15** and
[main at b7d4a4f5dd45e333add856a5ae9057fc5a57e93a](https://github.com/kyverno/chainsaw/tree/b7d4a4f5dd45e333add856a5ae9057fc5a57e93a).
Source and test links below are pinned to v0.2.15; fix links use merge SHAs and
adopter links use repository commit SHAs. Issue/comment links preserve the
specific report or discussion, but their open/closed state can change.

This is a curated decision ledger, not a list of every upstream issue. **Source
inspection is not runtime verification:** the linked upstream tests and adopter
examples were inspected. Their links do not establish that a project's suite
passes, and no remote CI or adopter runtime result is claimed here.

- **Semantics:** enduring behavior confirmed in the pinned implementation or
  maintainer explanation; revisit when changing the Chainsaw version.
- **Corrected historical bug:** merged correction present in the baseline;
  retain a regression concern, not a claim that the old defect still exists.
- **Unresolved report/request:** user evidence or a proposal, not a guaranteed
  reproduction or an available API.
- **Engineering inference:** a prevention decision drawn from the behavior;
  validate it against the actual resource/dependency boundary.
- **Adopter pattern:** observed repository practice, not a universal requirement.

## Assertions and execution semantics

Implementation guidance: [assertions](assertions.md) and
[integrations](integrations.md).

| Rule and failure mechanism | Prevention | Primary evidence and status |
| --- | --- | --- |
| An `assert` mismatch is retryable; an expression evaluation error can terminate immediately. Functions receiving a missing field as null can produce a type error rather than a normal mismatch. | Use a field projection or a type-appropriate fallback that still fails the intended readiness predicate. Test missing-field and ready cases; increasing a timeout cannot repair an invalid expression. | **Semantics, v0.2.15:** [assert implementation L84](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/assert/operation.go#L84), [bad-expression test L122](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/assert/operation_test.go#L122). **Maintainer workaround and user confirmation:** [#1823 fallback](https://github.com/kyverno/chainsaw/issues/1823#issuecomment-2264821533), [field projection](https://github.com/kyverno/chainsaw/issues/1823#issuecomment-2264825070), [confirmation](https://github.com/kyverno/chainsaw/issues/1823#issuecomment-2264835737). |
| A list-based resource assertion succeeds when **one** candidate matches; it does not prove every selected resource is ready. | Assert exact identities for required resources, or explicitly check a collected list's cardinality and membership before checking each member. | **Semantics, v0.2.15:** [candidate loop L112](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/assert/operation.go#L112), [list test L182](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/assert/operation_test.go#L182). |
| Iteration over an empty/null selection has no member that can fail. Iterating over objects that happen to exist does not prove that required objects exist. | Add a nonempty/exact-length assertion and required identities. A snapshot from a command is not automatically refreshed by retrying its later assertion. | **Maintainer semantics:** [#955 empty iteration](https://github.com/kyverno/chainsaw/issues/955#issuecomment-1960298200). **v0.2.15 example:** [array iteration](https://github.com/kyverno/chainsaw/blob/v0.2.15/testdata/e2e/examples/array-assertion/assertions.yaml). **Unresolved membership concern:** [#1143](https://github.com/kyverno/chainsaw/issues/1143#issuecomment-3738760946). Cardinality/membership safeguards are **engineering inference**, not proof that an object-list loop feature exists. |
| `error` succeeds at the first observation with no matching candidate, including absence or different fields. It neither submits an admission request nor proves continuous absence over an interval. | For absence, use only the intended resource identity; fields can turn an existing object into a non-match. For rejection, check the actual mutation error and intended reason. For sustained absence, define a bounded repeated observation with explicit scope. | **Semantics, v0.2.15:** [error operation L74](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/error/operation.go#L74), [absence test L42](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/error/operation_test.go#L42). Admission and interval limitations follow from this read/check implementation. |
| Supplying a `command`/`script` `check` replaces the default execution-error result. An output-only check can accept a failed command. | For expected success, include `($error == null): true` alongside the output check. For expected failure, require the intended error/output, not any nonzero result. | **Semantics, v0.2.15:** [command result L141](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/command/operation.go#L141), [command check test L83](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/command/operation_test.go#L83), [script check test L78](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/script/operation_test.go#L78). |
| Context selection precedes evaluation of bindings in that scope; inherited values or template defaults can be mistaken for invocation inputs. | Select the intended cluster/context at the scope that evaluates its bindings, and pass template inputs explicitly. Verify the resolved input rather than assuming an override. | **Semantics, v0.2.15:** [context then bindings L71](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/context/legacy.go#L71), [template override example](https://github.com/kyverno/chainsaw/blob/v0.2.15/testdata/e2e/examples/step-template-bindings/chainsaw-test.yaml). Historical fix is recorded separately below. |
| Operation outputs are added to a step's context; the runner does not return that context to the next step. A cross-step reference can be missing or resolve to an outer binding with the same name. | Keep consumers of a captured output in the same step or explicitly reacquire inputs at the required scope. Keep state needing refresh inside live assertions. | **Semantics, v0.2.15:** [step context creation L184](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L184), [operation output assignment L346](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L346), [same-step output example](https://github.com/kyverno/chainsaw/blob/v0.2.15/website/docs/general/outputs.md). |
| Report writers do not create the parent directory. Tests can pass while the runner exits unsuccessfully on report writing in a clean checkout. | Create the configured report directory before running and keep report failures visible in CI. | **Semantics, v0.2.15:** [JUNIT-TEST writer L65](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/report/junit.go#L65). **Local runtime:** missing-directory failure and successful report writing after directory creation were reproduced on the baseline. |

## Lifecycle and isolation semantics

Implementation guidance: [lifecycle and isolation](lifecycle-and-isolation.md).

| Rule and failure mechanism | Prevention | Primary evidence and status |
| --- | --- | --- |
| Independent tests do not form a filename-ordered workflow; teardown in another test can race setup or observation. | Put dependent actions in one Test's ordered steps. External invocation ordering needs its own failure/teardown owner. | **Maintainer semantics:** [#1133](https://github.com/kyverno/chainsaw/issues/1133#issuecomment-2066333334). **v0.2.15 source:** [ordered steps L184](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L184). |
| Random test namespaces isolate namespaced identities only. Literal namespaces bypass injection; pre-existing namespaces remain externally owned. | Keep ephemeral namespaces by default. Make fixed namespace ownership explicit and separately scope globals and cleanup. | **Semantics, v0.2.15:** [random namespace L172](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L172), [namespace creation/ownership L495](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L495), [namespacer L24](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/namespacer/namespacer.go#L24). Unique global identities and serialization only for actual conflicts are **engineering inference**. |
| `apply` patches existing objects without registering them for automatic deletion; successful apply does not transfer ownership or restore previous values. | Prefer fresh owned fixtures. For intentional updates, explicitly restore or remove only what the test owns and verify the final state. | **Semantics, v0.2.15:** [apply L103](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/apply/operation.go#L103), [create-path test L273](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/apply/operation_test.go#L273). **Maintainer confirmation:** [#2351](https://github.com/kyverno/chainsaw/issues/2351#issuecomment-3643942734). Source, not the test's title alone, establishes cleanup registration. |
| `finally` is per-step; `cleanup` is test-end. Step callbacks and native cleanup entries run in reverse registration order; native deletion precedes that step's explicit cleanup. Removing controllers too early can strand finalizers. | Collect diagnostics in catch; establish dependencies earlier than their consumers. Keep deletion dependencies alive until consumers are gone; explicitly handle resources created by shell commands. | **Semantics, v0.2.15:** [step cleanup/finally/catch L275](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L275), [reverse native entries L63](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/cleanup/cleaner/cleaner.go#L63), [cleanup documentation](https://github.com/kyverno/chainsaw/blob/v0.2.15/website/docs/step/cleanup.md). Finalizer-safe dependency ordering is **engineering inference**. |
| Debug deletion options weaken cleanup evidence. `skipDelete` does not imply every explicit teardown is disabled; fast namespace deletion does not prove completion. | Require a reason, bounded owner/deadline, and evidence of retained state or eventual removal. Inspect explicit cleanup before promising retention. | **Semantics, v0.2.15:** [skipDelete L551](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L551), [namespace wait selection L181](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L181), [option definitions](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/apis/v1alpha1/test.go). Bounded ownership is **engineering inference**. |
| Namespace setup runs for the primary selected cluster, not automatically for every later cluster target. | Declare and clean up secondary-cluster namespaces explicitly; route cluster/credential details to integrations. | **Semantics, v0.2.15:** [initial setup L172](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L172), [setup selects current client L490](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L490), [dynamic-clusters example](https://github.com/kyverno/chainsaw/blob/v0.2.15/testdata/e2e/examples/dynamic-clusters/chainsaw-test.yaml). **Historical discussion:** [#1535](https://github.com/kyverno/chainsaw/issues/1535#issuecomment-2159267551); not evidence of automatic cross-cluster provisioning. |

## Corrected historical bugs

All five corrections below are present in the v0.2.15 baseline. These are source
and regression-test inspections, not test runs in this checkout.

| Earlier failure mechanism | Prevention at this baseline | Pinned correction and evidence |
| --- | --- | --- |
| Retry handling could delay permanent create/patch/update errors or mishandle retryable failures. | Use operation-specific expectations and budgets; do not infer a universal retry policy for apply/assert from this fix. | **Corrected:** [#2342, 515e40772f3157866ec7759e4b747f26c5063a70](https://github.com/kyverno/chainsaw/commit/515e40772f3157866ec7759e4b747f26c5063a70), including create/patch/update retry tests. Apply's retry path is outside this correction. |
| Step-template default bindings overrode values supplied through `use.with.bindings`. | Pass invocation inputs explicitly; retain an override regression check when upgrading. | **Corrected:** [#2174, 3a833705bea195c5301db16c26a68a9d5670e0ee](https://github.com/kyverno/chainsaw/commit/3a833705bea195c5301db16c26a68a9d5670e0ee), with the pinned [override example](https://github.com/kyverno/chainsaw/blob/v0.2.15/testdata/e2e/examples/step-template-bindings/chainsaw-test.yaml). |
| Namespace teardown could occur before step cleanup. | Use current test-end cleanup semantics, while still ordering controller/finalizer dependencies deliberately. | **Corrected:** [#2437, 04cd02502e4d6b0b75121a8504fb6f6e4d8b57f6](https://github.com/kyverno/chainsaw/commit/04cd02502e4d6b0b75121a8504fb6f6e4d8b57f6); current [test cleanup registration L181](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/runner/runner.go#L181). |
| The summary could count a test as passing before its cleanup failed. | Treat cleanup failure as failure and retain logs/reports; do not normalize contradictory old summaries as expected behavior. | **Corrected:** [#2535, 3c476b05de2fe91209b0f837d6b33f9d432ca4a6](https://github.com/kyverno/chainsaw/commit/3c476b05de2fe91209b0f837d6b33f9d432ca4a6). **User confirmation on v0.2.14:** [#2498](https://github.com/kyverno/chainsaw/issues/2498#issuecomment-3647614689); this is reported user validation, not our runtime verification. |
| Resource-reference templating could lose non-string name/namespace/label assertion predicates. | Distinguish reference templating from assertion predicates; use the supported predicate structure rather than treating labels as arbitrary templated strings. | **Corrected:** [#2140, 2dc1aad8c3db883ce0f8d27d20e986ff727d47a7](https://github.com/kyverno/chainsaw/commit/2dc1aad8c3db883ce0f8d27d20e986ff727d47a7), with [label-predicate example](https://github.com/kyverno/chainsaw/blob/v0.2.15/testdata/e2e/examples/jmespath-label-condition/chainsaw-test.yaml). |

## Unresolved reports and proposed APIs

Statuses below describe the researched snapshot. Inspect the exact installed
version before attributing a new failure to one of these reports.

| Report or request and failure mechanism | Prevention | Evidence status and primary URL |
| --- | --- | --- |
| Negative `apply.expect` on a successful mutation can cause repeated patching while the expectation remains false. | Prefer a specific expected create rejection where suitable; prove the request and expected error. Keep negative mutation tests bounded and inspect API effects rather than retrying blindly. | **Unresolved user report on v0.2.12:** [#2442](https://github.com/kyverno/chainsaw/issues/2442). The v0.2.15 [apply retry/check path L86](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/apply/operation.go#L86) supports the risk mechanism; no reproduction or fix is claimed here. |
| A non-NotFound API error can stop assertion polling and a previous mismatch can mask the terminating error. | Preserve raw diagnostics and timestamps; inspect authentication/API failures and the live object before increasing timeouts or blaming the controller. | **Unresolved user report on v0.2.15/main:** [#2758](https://github.com/kyverno/chainsaw/issues/2758). **Source-supported mechanism:** [assert error handling L98](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/assert/operation.go#L98) and [last-error reporting L128](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/assert/operation.go#L128). Reported production timing is not independently verified. |
| Interrupting the runner can leave resources without executing normal cleanup. | Maintain run-specific ownership and an external recovery path; do not promise signal-safe cleanup or recovery of globals through namespace deletion. | **Unresolved interruption request:** [#2572](https://github.com/kyverno/chainsaw/issues/2572), [maintainer response](https://github.com/kyverno/chainsaw/issues/2572#issuecomment-3646431516). A promised future release is not baseline proof. |
| Generated subprocess kubeconfig can omit a user's Teleport-specific auth setup. | Verify native operations and subprocess helpers separately; keep any workaround scoped to the intended cluster and document credential provenance. Do not invent an injection opt-out field. | **Unresolved user report/feature request:** [#2751](https://github.com/kyverno/chainsaw/issues/2751). **v0.2.15 source:** [script kubeconfig injection L78](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/script/operation.go#L78). No Teleport reproduction is claimed here. |
| Proposed concurrency groups and command/script `poll` settings can be mistaken for supported YAML. | Use supported concurrency controls for actual conflicts. For external eventual state, use a bounded observation loop only when needed and safe to repeat. | **Feature requests, not v0.2.15 APIs:** [#2674 concurrency groups](https://github.com/kyverno/chainsaw/issues/2674), [#2772 command/script polling](https://github.com/kyverno/chainsaw/issues/2772). Existing one-execution command checks are shown in [command L123](https://github.com/kyverno/chainsaw/blob/v0.2.15/pkg/engine/operations/command/operation.go#L123). |

## Adopter patterns

These pinned repositories demonstrate practical choices. Copy the decision
pattern only when its resource lifecycle and environment match the task; their
timeouts, selectors, serialism, and installed controllers are not requirements
for other projects.

| Observed pattern | Why it is useful | Pinned evidence and limits |
| --- | --- | --- |
| OpenTelemetry Operator separates operation budgets and configures common failure collectors. | Long reconciliation/deletion may need different budgets from apply/exec. Namespace events and descriptions can explain failures before a pod has logs. | **Adopter source inspection:** [.chainsaw.yaml L7–L34 at f88d3478e7abdb046e7252440c754592ba67a8dc](https://github.com/open-telemetry/opentelemetry-operator/blob/f88d3478e7abdb046e7252440c754592ba67a8dc/.chainsaw.yaml#L7-L34). No benchmark or runtime success is inferred. |
| Apache Spark Kubernetes Operator collects events/descriptions/logs in catch before a finally delete, and pins/verifies Chainsaw while running selected e2e groups serially. | Shows diagnostic-before-teardown ordering and reproducible binary selection. A project's shared default namespace can justify serialization; it does not justify serializing every independent suite. | **Adopter source inspection, e5c5731a10ec20dad9e0aa8dc79e33b4c8bb5c4f:** [catch/finally L51–L84](https://github.com/apache/spark-kubernetes-operator/blob/e5c5731a10ec20dad9e0aa8dc79e33b4c8bb5c4f/tests/e2e/python/chainsaw-test.yaml#L51-L84), [v0.2.15 pin L9–L11](https://github.com/apache/spark-kubernetes-operator/blob/e5c5731a10ec20dad9e0aa8dc79e33b4c8bb5c4f/.github/workflows/build_and_test.yml#L9-L11), [checksum check L191–L197](https://github.com/apache/spark-kubernetes-operator/blob/e5c5731a10ec20dad9e0aa8dc79e33b4c8bb5c4f/.github/workflows/build_and_test.yml#L191-L197), [serial invocation L230](https://github.com/apache/spark-kubernetes-operator/blob/e5c5731a10ec20dad9e0aa8dc79e33b4c8bb5c4f/.github/workflows/build_and_test.yml#L230). This records the pattern, not an endorsement of every shell detail. |

## Local validation of skill examples

On 2026-10-13, the SHA256-verified v0.2.15 release binary was used on Linux amd64
with a disposable single-node Kind v0.33.0 cluster. This validation is separate
from inspection of upstream regression tests and adopter repositories.

- All four complete YAML examples passed `lint test` or `lint configuration`.
  The exact `exact-membership` and `ordered-configmap` Tests passed on Kind;
  `process-results` passed with `--no-cluster`. The Configuration produced valid
  JUnit XML when its report directory existed.
- Controlled counterexamples rejected expected stdout with a nonzero exit,
  missing collection members, wrong values, and an absent token that never
  appeared. A delayed token passed through the retrying field-local predicate.
  Unnamed ANY matching, empty iteration, and stdout-only checks reproduced the
  false-pass mechanisms described above.
- Concurrent tests used distinct ephemeral namespaces. Owned-resource cleanup
  worked after success and failure, catch diagnostics were retained, and borrowed
  objects retained their UID and patched state. Output bindings were available
  within a step and unavailable in the next. The owned cluster was deleted and
  its absence verified.
- Independent authoring, review, and diagnosis requests exercised reference
  selection and the integration boundary. The authoring and diagnosis YAML was
  schema-linted; the review identified four deliberately introduced false-pass
  and ownership hazards. These checks do not validate the fictional controller
  or service used in those requests.

The server was Kubernetes v1.37.0 and kubectl was v1.35.9, outside the supported
minor-version skew; targeted calls succeeded. Real controllers/admission rules,
multi-cluster execution, external authentication plugins, interruption recovery,
and remote CI were not exercised. Admission and StepTemplate fragments retain
external prerequisites. A successful example does not establish those integrations.
