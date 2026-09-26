# Status and Conditions

Kubebuilder/Go implementation guide for status and conditions in CRD types.

For canonical Kubernetes API conventions on conditions, see:
[K8s API conventions — typical status properties](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties).

## Baseline Status Shape

When designing status for a long-running reconciled resource, include:

- `observedGeneration` (controller-owned) — tracks which spec generation was last processed
- `conditions` as `[]metav1.Condition` (controller-owned) — standard status reporting

### Example

```go
import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

type FooStatus struct {
    // ObservedGeneration is the .metadata.generation last observed by the controller.
    //
    // +kubebuilder:validation:Minimum=0
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Conditions represents the latest available observations of the resource's state.
    //
    // +listType=map
    // +listMapKey=type
    // +kubebuilder:validation:MaxItems=20
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

Notes:

- The controller should set `status.observedGeneration` when it updates status
- Bound conditions array with `MaxItems` to prevent unbounded growth
- Use `+listType=map` / `+listMapKey=type` for SSA correctness

## Enable Status Subresource

The root type **must** enable the `/status` subresource:

```go
// +kubebuilder:subresource:status
// +kubebuilder:object:root=true
type Foo struct {
    // ...
}
```

This ensures status updates use the correct endpoint and RBAC boundary.

## Standard Condition Types

Follow Kubernetes conventions for condition naming:

| Type | Meaning |
|------|---------|
| `Ready` | Resource is fully operational |
| `Available` | Resource is fully functional |
| `Progressing` | Resource is being created or updated |
| `Degraded` | Resource failed to reach or maintain desired state |
| `Reconciling` | Controller is actively working on the resource |

Prefer positive polarity (`Ready=True` means good) except for `Degraded`.

## Setting Conditions in Controller

Since `k8s.io/apimachinery` v0.29+, `meta.SetStatusCondition` returns a `bool`:
`true` if the condition was actually modified (new, status/reason/message/observedGeneration changed),
`false` if nothing changed. Use this return value to avoid unnecessary status API calls.

```go
import "k8s.io/apimachinery/pkg/api/meta"

changed := meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
    Type:               "Ready",
    Status:             metav1.ConditionTrue,
    Reason:             "ReconcileSuccess",
    Message:            "All resources are ready",
    ObservedGeneration: foo.Generation,
})
// changed == true  → condition was added or modified, status update needed
// changed == false → no-op, skip the API call
```

Rules:

- Always set `ObservedGeneration` on conditions
- Use `Reason` as PascalCase machine-readable token
- Use `Message` as human-readable description
- `LastTransitionTime` is set automatically by `SetStatusCondition`

## Printer Columns for Status

Surface readiness in `kubectl get` output:

```go
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=".status.conditions[?(@.type=='Ready')].status"
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=".metadata.creationTimestamp"
```

## Status Update Pattern in Controller

Use `SetStatusCondition`'s bool return to track whether an update is needed.
Always update status **explicitly** at the end of `Reconcile` so errors propagate
and trigger a proper requeue.

**Do not use `defer` for status updates** — it swallows `Status().Update()` errors,
preventing the controller from requeueing on failure.

### Recommended: inline status update at end of Reconcile

```go
func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    var foo cachev1.Foo
    if err := r.Get(ctx, req.NamespacedName, &foo); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // --- reconciliation logic ---
    result, reconcileErr := r.reconcile(ctx, &foo)

    // Set condition and update status inline — only write if something changed
    foo.Status.ObservedGeneration = foo.Generation
    condition := metav1.Condition{
      Type: "Ready",
      ObservedGeneration: foo.Generation
    }

    if reconcileErr != nil {
        condition.Status = metav1.ConditionFalse
        condition.Reason = "ReconcileError"
        condition.Message = reconcileErr.Error()
    } else {
        condition.Status = metav1.ConditionTrue
        condition.Reason = "ReconcileSuccess"
        condition.Message = "All resources are ready"
    }

    if meta.SetStatusCondition(&foo.Status.Conditions, condition) {
        if err := r.Status().Update(ctx, &foo); err != nil {
            logger.Error(err, "Failed to update status")
            return ctrl.Result{}, err
        }
    }

    return result, reconcileErr
}
```

### Why NOT defer

| Problem with `defer` | Impact |
|----------------------|--------|
| Swallows `Status().Update()` errors | Controller won't requeue on status write failure |
| Can't influence return value | Named returns make it possible but fragile and hard to read |
| Hides control flow | Two error paths (reconcile + status) mixed in non-obvious order |
| Status update races | If reconcile returns an error AND status update fails, only the reconcile error surfaces |

### Why this pattern

| Concern | Solution |
|---------|----------|
| Unnecessary API calls | `SetStatusCondition` returns `false` when nothing changed → skip `Update()` |
| Status update failure | Error returned explicitly → triggers requeue with backoff |
| No tracking variable | Single `if SetStatusCondition(...) { Update() }` — no `statusChanged` bool to maintain |
| Multiple conditions | Call `SetStatusCondition` for each condition in a separate statement, then combine the returned booleans with logical OR |

### Multiple conditions: update after each action

Update status immediately after each reconciliation step rather than batching at
the end. This gives real-time observability via `kubectl get` and preserves
progress if the controller crashes mid-reconcile.

```go
func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    var foo cachev1.Foo
    if err := r.Get(ctx, req.NamespacedName, &foo); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    foo.Status.ObservedGeneration = foo.Generation

    // Step 1: ensure Deployment
    if err := r.ensureDeployment(ctx, &foo); err != nil {
        if meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
            Type: "Ready", Status: metav1.ConditionFalse,
            Reason: "DeploymentFailed", Message: err.Error(),
            ObservedGeneration: foo.Generation,
        }) {
            _ = r.Status().Update(ctx, &foo) // best-effort before returning error
        }
        return ctrl.Result{}, err
    }
    if meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
        Type: "Progressing", Status: metav1.ConditionTrue,
        Reason: "DeploymentReady", Message: "Deployment is up",
        ObservedGeneration: foo.Generation,
    }) {
        if err := r.Status().Update(ctx, &foo); err != nil {
            return ctrl.Result{}, err
        }
    }

    // Step 2: ensure Service
    if err := r.ensureService(ctx, &foo); err != nil {
        if meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
            Type: "Ready", Status: metav1.ConditionFalse,
            Reason: "ServiceFailed", Message: err.Error(),
            ObservedGeneration: foo.Generation,
        }) {
            _ = r.Status().Update(ctx, &foo)
        }
        return ctrl.Result{}, err
    }

    // Final: all steps done
    if meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
        Type: "Ready", Status: metav1.ConditionTrue,
        Reason: "ReconcileSuccess", Message: "All resources are ready",
        ObservedGeneration: foo.Generation,
    }) {
        if err := r.Status().Update(ctx, &foo); err != nil {
            return ctrl.Result{}, err
        }
    }

    return ctrl.Result{}, nil
}
```

| Tradeoff | Batch at end | Update after each step |
|----------|-------------|----------------------|
| API calls | 1 | N (one per step) |
| Observability | All-or-nothing | Real-time progress |
| Crash resilience | Status stale if crash mid-reconcile | Status reflects last completed step |
| Debugging | Hard to tell where it's stuck | `kubectl get` shows exactly which step |

**Prefer update-after-each-step** for operators with multiple phases. The extra
API calls are negligible compared to the value of knowing where reconciliation
is at any moment.

### Alternative: MergeFrom patch (lowest conflict risk)

When multiple controllers or webhooks may touch status concurrently, use a merge
patch instead of a full update:

```go
if statusChanged {
    basePatch := client.MergeFrom(foo.DeepCopy())
    if err := r.Status().Patch(ctx, &foo, basePatch); err != nil {
        logger.Error(err, "Failed to patch status")
        return ctrl.Result{}, err
    }
}
```

### Required imports

```go
import (
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
)
```

## Generate and Verify

After adjusting types/markers:

1. Run `make generate` and `make manifests`
2. Inspect the generated CRD YAML and confirm:
   - `spec.versions[*].schema.openAPIV3Schema.properties.status.properties.conditions` exists
   - `x-kubernetes-list-type: map` and `x-kubernetes-list-map-keys: ["type"]` present
