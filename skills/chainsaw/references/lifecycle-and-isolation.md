# Lifecycle and isolation

Use this reference when a test changes shared state, depends on earlier actions,
or leaves resources behind. The baseline is Chainsaw **v0.2.15**; the
[evidence ledger](evidence.md) records source inspection and its limits.

## Put dependencies in one Test

Represent a dependent flow as ordered `spec.steps` in one `Test`: create the
input, observe the outcome, then exercise deletion. Each step's `try` operations
also run in order. Separate tests are independent and may run concurrently;
numeric directory or file prefixes do not establish a dependency graph. This is
[confirmed by the maintainer](https://github.com/kyverno/chainsaw/issues/1133#issuecomment-2066333334).

Use separate tests when each can establish its own prerequisites. If an external
workflow genuinely needs distinct invocations, let that workflow enforce the
order and own teardown on failure; record the dependency and demonstrate that
the later invocation cannot run after failed setup. Serial execution alone does
not make one test depend on another.

For reusable steps, pass inputs explicitly through `use.with.bindings` rather
than depending on another test's state. See [assertions](assertions.md) for
matching and expression behavior, and [integrations](integrations.md) for bindings,
templates, and external-tool wiring.

### A complete ordered example

Save this as `ordered-configmap/chainsaw-test.yaml` and run
`chainsaw test --test-dir ./ordered-configmap` against the intended cluster using
v0.2.15. It requires permission to create/delete a test namespace and ConfigMaps;
the `events` collector also requires `kubectl` and permission to read events.
Keep the default random namespace and cleanup enabled. No operator or fixture
files are needed. Success means all three steps and cleanup pass.

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: ordered-configmap
spec:
  catch:
    - events: {}
  steps:
    - name: create-input
      try:
        - apply:
            resource:
              apiVersion: v1
              kind: ConfigMap
              metadata:
                name: ordered-input
              data:
                state: prepared
    - name: observe-input
      try:
        - assert:
            resource:
              apiVersion: v1
              kind: ConfigMap
              metadata:
                name: ordered-input
              data:
                state: prepared
    - name: delete-input
      try:
        - delete:
            ref:
              apiVersion: v1
              kind: ConfigMap
              name: ordered-input
```

Omitting `metadata.namespace` lets Chainsaw inject the test namespace into the
namespaced resource. Automatic cleanup remains a fallback if an earlier step
fails; the explicit delete tests the normal deletion path. This complete example
passed schema lint and execution on disposable Kind with v0.2.15. See the
[validation record](evidence.md#local-validation-of-skill-examples) for scope and limits.

## Choose an ownership boundary

| Situation | Default and reason | Exception control and required evidence |
| --- | --- | --- |
| Independent namespaced test | Keep the random ephemeral namespace; omit literal namespaces in native resource operations so injection follows the test. | A fixed namespace must serve a real integration requirement. Inventory existing objects, constrain names/selectors to the test, and show that teardown cannot delete unrelated objects. |
| Pre-existing namespace | Treat it as externally owned. Chainsaw reuses it and does not automatically delete it. | If the namespace is dedicated to the run, assign an explicit external cleanup owner and verify its contents before deletion. Never infer ownership from its name alone. |
| Pre-existing resource | Prefer a new test-owned resource. `apply` patches an existing object but only registers a newly created object for automatic cleanup. | For an intentional update, identify the owner and establish explicit restoration or removal semantics. Verify the post-test state; automatic cleanup does not roll back a patch. |
| Resource with a literal namespace | Treat that namespace as an explicit target; injection does not replace it. | Retain the literal only when the scenario requires it, and prove the resource and cleanup selectors belong to this run. |
| Cluster-scoped resource or global effect | Give each run a unique identity where the API allows it; keep cleanup equally specific. A random namespace does not isolate CRDs, cluster roles, webhooks, shared controller settings, or external systems. | When a global identity or setting cannot be separated, serialize the tests/jobs that actually conflict, or use a separately owned test environment. Record the conflicting object/setting and check restoration. |

Use existing test/run bindings to derive unique cluster-scoped names when
supported; see [integrations](integrations.md) for bindings. A unique name is not
enough if a webhook, selector, or controller still affects other tests. Namespace
templates can supply required labels, but do not add a template just for naming.

Keep independent tests concurrent. For a proven shared conflict, use the
supported `spec.concurrent: false` or the appropriate invocation's `--parallel
1`; neither coordinates separate CI jobs or separate Chainsaw processes. Those
need orchestration at the owner of the shared resource. The proposed concurrent
groups in [#2674](https://github.com/kyverno/chainsaw/issues/2674) are not a
v0.2.15 API. Serializing an entire suite requires evidence that the conflict
actually spans the entire suite.

For a secondary cluster, explicitly account for namespace creation and deletion
there. Initial namespace setup targets the primary selected cluster; do not
assume that namespace selection provisions every cluster. Continue with
[integrations](integrations.md) for cluster selection, subprocess credentials,
and multi-cluster handoff.

## Collect evidence before deleting it

Use `catch` for failure diagnostics while the relevant objects still exist.
Collect namespace events first, then resource descriptions and controller or
workload logs relevant to the failure. Scope collectors to the affected namespace
and identities. Test/configuration catch handlers can provide a shared baseline;
step handlers add scenario-specific evidence. An assertion diff is not a live
snapshot and may obscure an API failure; retain the full object and execution
logs when needed. See [assertions](assertions.md) for assertion failure diagnosis.

Choose teardown by the lifetime of the dependency:

| Mechanism | When it runs | Use it for |
| --- | --- | --- |
| `catch` | Failure handling before `finally` and test-end cleanup. | Diagnostics, not routine teardown. |
| `finally` | At the end of that step, including when it fails. | Temporary state no later step needs. It is too early for a controller needed by later steps or finalizers. |
| Step `cleanup` | At test end, with step cleanup callbacks in reverse registration order. | Shell-created resources and dependencies that must outlive later steps. Operations within one block retain their declared order. |
| Automatic resource cleanup | At test end; resources created by native operations are deleted in reverse registration order. | Test-created native resources. Within a step callback, this runs **before** that step's explicit `cleanup` block. |
| Test namespace cleanup | After step cleanup callbacks. | A namespace Chainsaw created, provided automatic deletion is enabled. |

Establish a controller dependency in an earlier step and create its custom
resources in later steps so their cleanup can complete first. Keep the controller,
its credentials, and any webhook needed for deletion alive until dependent
resources are actually gone. If dependency order differs from creation order,
use deliberate explicit teardown and verify deletion before removing the
dependency. Do not remove finalizers to make a test appear to pass; diagnose the
controller or teardown ordering first.

Commands and scripts do not register their created resources with Chainsaw's
native cleaner. Pair their mutations with an explicit, bounded cleanup in the
same ownership scope, including local temporary files or external state. Use
`finally` only for effects confined to the step; otherwise register `cleanup`
where the dependency is established. Make teardown tolerate an already absent
owned resource without suppressing authentication, permission, or deletion
errors. Shell command/check behavior is covered in [integrations](integrations.md).

## Bound debug exceptions and interruption recovery

Keep `skipDelete` and `fastNamespaceDeletion` disabled for normal runs.

- Enable `skipDelete` only to retain evidence for a specific debugging run.
  Record cluster, namespace, owned identities, the person/job responsible for
  removal, and a cleanup deadline. Verify the retained state and its eventual
  removal. It does not automatically disable explicit `delete`, `finally`, or
  custom `cleanup` operations; inspect those before claiming retention.
- Enable `fastNamespaceDeletion` only when intentionally accepting asynchronous
  namespace deletion. It skips waiting for namespace deletion; it does not prove
  the namespace is gone or repair finalizers. Bound the namespace's ownership
  and verify eventual removal outside the run before reusing a fixed identity.
- Treat interruption, forced termination, and runner loss as possible cleanup
  bypasses. [#2572](https://github.com/kyverno/chainsaw/issues/2572#issuecomment-3646431516)
  discusses interruption handling; it is not a guarantee for the pinned baseline.
  Record the run's owned namespaces and cluster-scoped resources before relying
  on an external recovery job. Recovery must select only that run's resources
  and confirm they are gone; namespace deletion alone cannot recover global
  resources or external side effects.

Keep cleanup errors visible in the result. A historically incorrect passing
summary was corrected in v0.2.14; do not use that old behavior as a reason to
ignore failed teardown. The [ledger](evidence.md) separates the corrected bug
from current ownership limits and unresolved reports.
