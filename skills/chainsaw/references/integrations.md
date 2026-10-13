# Integrate Chainsaw with the project's tools

Read this reference for version/configuration selection, bindings, cluster targeting,
and local or CI execution. Reuse project targets and specialized tooling; keep
environment provisioning in its own workflow.

## Version and configuration

Run `chainsaw version` and inspect the project's install target, lock/pin, CI inputs,
and CLI help. Kyverno Chainsaw's CLI includes `test` and `lint`; an unrelated
Homebrew tool also uses the name Chainsaw. Use the Kyverno-qualified package or
official release described in the [installation documentation](https://kyverno.github.io/chainsaw/0.2.15/quick-start/install/).

Match local and CI versions. Reference examples use v0.2.15; assess an older or newer
project explicitly instead of silently replacing its binary or schema. A merged
fix does not establish the first release containing it without additional evidence.

Chainsaw uses an explicitly supplied `--config` file, otherwise `.chainsaw.yaml`
in the current working directory, otherwise embedded defaults. Make configuration
and test directories deterministic in CI and project targets. Do not assume running
the same command from another directory uses the same configuration.

This complete v1alpha2 Configuration illustrates operation-specific budgets and
reports; adjust the budgets to the controller's measured convergence and cleanup:

Create the report directory before running, for example with
`mkdir -p artifacts/chainsaw` in a clean checkout. Chainsaw does not create it.

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha2
kind: Configuration
metadata:
  name: test-suite
spec:
  timeouts:
    apply: 10s
    assert: 1m
    cleanup: 1m
    delete: 30s
    error: 30s
    exec: 10s
  report:
    format: JUNIT-TEST
    path: artifacts/chainsaw
    name: results
```

Treat these durations as an example, not universal defaults. Override a slow
boundary at the narrowest useful operation/step/test scope; do not inflate every
timeout to hide a race, invalid expression, or missing prerequisite.

## Bindings, outputs, and reusable steps

Bindings are scoped: a lower-scope binding can hide an outer one and the outer value
becomes visible again when that scope ends. Keep names meaningful and avoid hiding
built-ins. Resource values can use expressions; command/script content is not
resource templating. Pass computed values through explicit `env` bindings.

Commands do environment-variable expansion but do not run arguments through shell
expansion. Use `command` for an executable plus arguments; use `script` when shell
syntax is actually needed. The default script shell is `sh`; Bash-only options such
as `pipefail` require an explicit compatible shell. Preserve an earlier failure in
multi-command scripts and pipelines rather than returning only the last command's
status. Respect the test-relative working directory or set `workDir` intentionally.

Outputs are evaluated after their operation and made available to later operations
in the same step; they are not exported to subsequent steps in v0.2.15. Keep
consumers of a captured identity in that step, or explicitly reacquire the input
at the scope that needs it. Outputs are snapshots, not subscriptions to live state:
keep state that must refresh inside the retrying assertion.

Under `spec.steps`, pass template parameters through the supported call-site
override:

```yaml
- use:
    template: template.yaml
    with:
      bindings:
      - name: input
        value: from-test
```

This step fragment requires an existing StepTemplate accepting `input`; `use`
does not belong inside a step's `try` list.
The precedence correction and regression are linked in [evidence.md](evidence.md).
At a scope, cluster/context selection occurs before that scope's own bindings are
installed. Put required selection values in an earlier available scope, such as
scenario bindings for a test-level selection. Verify scope when moving an expression.

## Cluster and identity selection

Use the project's explicit kubeconfig/context or registered Chainsaw cluster.
Avoid scripts that mutate the user's ambient kubeconfig or current context. For
identity tests, register the intended contexts and select them per operation rather
than repeatedly switching a shared kubeconfig.

With cluster execution enabled, Chainsaw builds a temporary subprocess kubeconfig
for the selected target; its single context is named `chainsaw`. Do not require an
original context name that is absent from that generated file. Check both native
operations and kubectl-backed helpers when authentication uses an external plugin.
An unresolved auth-provider report is a diagnostic lead, not permission to weaken
authentication or invent configuration fields.

The test's ephemeral namespace is established on its selected cluster. Switching
the step's cluster does not automatically create the same namespace there. Establish
and clean up owned test namespaces on each secondary cluster with native operations.
Do not copy server-owned Namespace metadata or use a shared temporary kubeconfig
filename across concurrent tests. See [lifecycle and isolation](lifecycle-and-isolation.md)
for ownership and teardown boundaries.

## Local tooling: integration advice only

Prefer the existing local test target. If cluster setup is missing, recommend an
available specialized skill whose publisher and scope were verified, or point to
the [official Kind documentation](https://kind.sigs.k8s.io/docs/user/quick-start/).
Do not label an unverified third-party skill “official,” hard-require another skill,
install one automatically, or reproduce a cluster/bootstrap tutorial here.

Check the boundaries that can prevent a Chainsaw run from exercising its contract:

- The selected kubeconfig reaches the intended test cluster from the process or
  container running Chainsaw; credentials and authentication plugins remain usable.
- The project's CRDs, controller, and required webhooks are available before test
  stimuli. Availability/readiness is observed, not inferred from installation success.
- A locally built controller image is accessible to that cluster, and the project's
  image/pull settings actually select it rather than an unrelated published build.
- Diagnostics can be collected before the external workflow destroys its cluster.

Delegate cluster provisioning, image loading, Helm/operator installation, and
general Kubernetes troubleshooting to those tools' workflows. Chainsaw remains
responsible for test selection, assertions, fixture ownership, and test outcomes.

## CI outcomes and diagnostics

Pin Chainsaw independently from an installer action's version. Use the project's
existing runner and artifact mechanisms; the [official installer](https://github.com/kyverno/action-install-chainsaw)
is an option when GitHub Actions installation is needed, not a required platform.

Ensure selection/sharding runs the intended tests, not an accidentally empty subset.
Give independent runs/shards distinct report locations so they do not overwrite one
another. Preserve the Chainsaw exit status through log piping and wrapper commands.
Upload structured reports and console/collector diagnostics even after a failed run;
do not turn reporting or teardown failures into a green job.

Use `catch` to collect events, descriptions/resource state, and relevant controller
logs before cleanup. A Pod that never started may have no logs, so logs alone are
insufficient. Avoid dumping Secret contents or sensitive command arguments into
artifacts; scope and suppress output where the fixture requires it.

Keep ordinary resource cleanup enabled. Retaining resources for diagnosis is a
deliberate, bounded debugging choice; distinguish it from the normal CI contract.
An outer ephemeral-cluster teardown can recover after cancellation, but does not
prove that a controller's resource/finalizer cleanup works. Report lint, clusterless
execution, cluster integration, and CI observations separately.
