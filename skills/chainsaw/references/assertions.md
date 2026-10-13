# Assertions that test the intended contract

Read this reference for resource matching, eventual state, collections, admission
denial, and command results. Examples use the v0.2.15 JMESPath assertion compiler;
check the project's compiler before transferring expressions to a CEL suite.
Source and version qualifications are collected in [evidence.md](evidence.md).

## Selection is part of the assertion

A named resource assertion retries NotFound and ordinary mismatches. An unnamed
assertion searches candidates and succeeds when **any one** matches. It does not
prove that all selected objects match, or that every expected object exists.

Use a name and the correct namespace when checking one identity. For generated
names, select through a test-specific label or ownership relationship and assert
the required membership/count separately. Do not select unrelated workloads from
all namespaces when the contract concerns one test's resources.

Compare stable behavior the controller owns. Avoid whole-object snapshots containing
server-generated timestamps, resource versions, unrelated defaults, or volatile
messages. Where a controller exposes generation tracking, ensure the observed
result belongs to the change being tested rather than an earlier reconciliation.

## Missing fields must remain retryable

Controller-populated fields may be absent at first. Passing `null` to a typed
function can cause an evaluation error that terminates polling. A larger timeout
does not make such an expression retryable.

This operation fragment requires the field before evaluating its string predicate:

```yaml
- assert:
    resource:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: result
      data:
        token:
          (regex_match('^ready-[a-z]+$', @)): true
```

In a controller test, `result` must be the expected controller-produced resource.
Validate the predicate against a missing field, a wrong value, and the intended
value. A type-appropriate fallback can also work, but it must fail the predicate
when the field is missing. Do not copy `field || ''` mechanically: JMESPath `||`
also replaces false booleans and empty strings, arrays, or objects; it preserves
numeric zero. An empty string may satisfy some predicates. See the
[JMESPath specification](https://jmespath.org/specification.html#or-expressions).

Keep live lookups inside `assert` expressions when they must refresh on each poll.
Bindings and command outputs evaluated earlier are snapshots; repeatedly asserting
the same captured value does not fetch a new one.

## Collection checks need membership

Iteration over an empty filtered array can pass without examining an element.
Check existence/membership first. Require exact cardinality only when the contract
forbids extra members; a contract requiring A and B may legitimately allow C.

This complete Test requires exactly two labeled ConfigMaps, both with `result: ready`.
The label filter excludes unrelated namespace objects, including Kubernetes-created
ConfigMaps. The API lookup is reevaluated inside the assertion.

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: exact-membership
spec:
  steps:
  - name: create-members
    try:
    - create:
        resource:
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: worker-a
            labels:
              fixture: membership
          data:
            result: ready
    - create:
        resource:
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: worker-b
            labels:
              fixture: membership
          data:
            result: ready
  - name: verify-members
    try:
    - assert:
        resource:
          (x_k8s_list($client, 'v1', 'ConfigMap', $namespace)):
            (length(items[?metadata.labels.fixture == 'membership'])): 2
            (sort(items[?metadata.labels.fixture == 'membership'].metadata.name)):
            - worker-a
            - worker-b
            ~.(items[?metadata.labels.fixture == 'membership']):
              data:
                result: ready
```

Removing worker-b or changing either member's value must fail. For multiple
individually named objects, separate named assertions are often simpler than a list
expression. This example tests assertion plumbing; a controller test must observe
its controller's outputs rather than only reasserting its own setup data.

## `error` is an inverse match, not an execution failure

For a resource assertion, `error` polls until no selected object matches:

| Observation | Result |
| --- | --- |
| Object absent or no candidates selected | Pass |
| Object exists but differs from the asserted fields | Pass |
| At least one selected object matches | Retry, then fail if it continues matching |
| Lookup fails for another reason, or an expression is invalid | Fail |

It succeeds at the first non-matching observation. It does not prove continued
absence for its entire timeout or that a required alternative state was reached.
Use a positive `assert` for that state. For identity absence, omit unrelated
spec/status fields so a field mismatch cannot stand in for deletion.

For admission rejection, use an operation's `expect.check`. Establish controller,
webhook, and policy prerequisites first. Prefer `create` when the stimulus is a
new-object submission: `apply` can retry an unexpected successful create as patches.
This is a default for that contract, not a universal ban on `apply` or `expect`.

The following operation fragment assumes the project's admission rule named
`require-owner-label` rejects this fixture; adapt the identity and diagnostic to
the actual rule. It is not a runnable admission test on an unconfigured cluster.

```yaml
- create:
    resource:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: rejected-fixture
    expect:
    - match:
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: rejected-fixture
      check:
        ($error != null): true
        (contains($error || '', 'require-owner-label')): true
```

An RBAC denial, connection failure, another policy's denial, and accidental
acceptance must fail. Match stable rule/reason fragments rather than an entire
server-formatted message. Add positive control cases when they establish that the
fixture and admission prerequisites actually exercise the intended rule.

## Custom process checks replace default success handling

A content-only `check` can accept expected stdout even when the process exits
nonzero. Require `$error == null` for success, and check the intended cause when
failure is expected. Prefer structured output over matching incidental prose.

This complete Test is suitable for a controlled `chainsaw test --no-cluster` run:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: process-results
spec:
  steps:
  - name: successful-command
    try:
    - script:
        content: printf 'done\n'
        check:
          ($error == null): true
          ($stdout): "done\n"
  - name: expected-rejection
    try:
    - script:
        content: |
          printf 'fixture: rejected input\n' >&2
          exit 1
        check:
          ($error != null): true
          "(contains($stderr, 'fixture: rejected input'))": true
```

As a counterexample, make the first script print the same stdout and exit 1: the
test must fail. These scripts validate process semantics, not Kubernetes admission.
Commands and scripts run once; an execution timeout does not cause repeated probes.
Use live resource assertions or an explicitly bounded, read-only polling probe for
external convergence. Do not invent polling fields from an open feature request.
