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

// Set True only after observing the readiness contract below.
changed := meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
    Type:               "Ready",
    Status:             metav1.ConditionTrue,
    Reason:             "DeploymentAvailable",
    Message:            "Deployment rollout is complete",
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

These examples assume one writer owns Foo status. Here `Ready=True` means the
required Service was ensured successfully and the managed Deployment's current
rollout is complete: its controller has observed the latest generation, all desired
replicas are updated and available, and no old replicas remain. A missing or deleting
Deployment is not ready. Zero replicas count as a completed scale-to-zero operation;
choose a different contract if the API requires a running workload. Additional
required dependencies must contribute their own observed readiness checks.

This follows the [Kubernetes v1.35.0 rollout-completion checks](https://github.com/kubernetes/kubernetes/blob/v1.35.0/pkg/controller/deployment/util/deployment_util.go#L687-L694).
`reconcile` must ensure the desired Deployment and Service before returning success.
Use `APIReader: mgr.GetAPIReader()` on the reconciler for this observation: a cached
read immediately after a write can still show the previous, healthy Deployment spec.
A nil error alone does not prove readiness. A delayed requeue can mean either waiting
for readiness or periodically checking an already healthy object; see the
[controller-runtime v0.23.0 Result contract](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.0/pkg/reconcile/reconcile.go).

```go
func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    var foo cachev1.Foo
    if err := r.Get(ctx, req.NamespacedName, &foo); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    if !foo.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, &foo)
    }
    // Include only when effects need controller-managed cleanup.
    if err := r.ensureFinalizer(ctx, &foo); err != nil {
        return ctrl.Result{}, err
    }

    result, reconcileErr := r.reconcile(ctx, &foo)
    if err := r.updateStatus(ctx, &foo, reconcileErr); err != nil {
        logger.Error(err, "Failed to update status")
        return ctrl.Result{}, err
    }
    return result, reconcileErr
}

func deploymentReady(dep *appsv1.Deployment) bool {
    if !dep.DeletionTimestamp.IsZero() {
        return false
    }
    desired := int32(1) // Deployment API default when replicas is omitted.
    if dep.Spec.Replicas != nil {
        desired = *dep.Spec.Replicas
    }
    return dep.Status.ObservedGeneration >= dep.Generation &&
        dep.Status.UpdatedReplicas == desired &&
        dep.Status.Replicas == desired &&
        dep.Status.AvailableReplicas == desired
}

func (r *FooReconciler) updateStatus(ctx context.Context, foo *cachev1.Foo, reconcileErr error) error {
    condition := metav1.Condition{
        Type: "Ready", Status: metav1.ConditionFalse,
        Reason: "DependencyNotReady", Message: "Waiting for Deployment rollout",
        ObservedGeneration: foo.Generation,
    }
    if reconcileErr != nil {
        condition.Reason = "ReconcileError"
        condition.Message = reconcileErr.Error()
    } else {
        var dep appsv1.Deployment
        if err := r.APIReader.Get(ctx, client.ObjectKeyFromObject(foo), &dep); err != nil {
            if !apierrors.IsNotFound(err) {
                return err
            }
        } else if deploymentReady(&dep) {
            condition.Status = metav1.ConditionTrue
            condition.Reason = "DeploymentAvailable"
            condition.Message = "Deployment rollout is complete"
        }
    }

    generationChanged := foo.Status.ObservedGeneration != foo.Generation
    foo.Status.ObservedGeneration = foo.Generation
    conditionChanged := meta.SetStatusCondition(&foo.Status.Conditions, condition)
    if generationChanged || conditionChanged {
        return r.Status().Update(ctx, foo)
    }
    return nil
}
```

`Owns(&appsv1.Deployment{})` must watch status changes; avoid predicates that discard
those events. If a dependency has no watch, return a suitable `RequeueAfter` while
waiting. The helper does not inspect the result delay, so a healthy periodic requeue
keeps `Ready=True`. To report successful reconciliation independently of availability,
use a separate condition such as `Reconciled`.

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
| Other status fields | Track their changes too; a condition no-op must not suppress an `observedGeneration` update |
| Multiple conditions | Call `SetStatusCondition` for each condition in a separate statement, then combine the returned booleans with logical OR |

### Multiple conditions: update after each action

When intermediate progress matters, persist a condition after a completed step.
Creation or a successful apply means the desired resource was ensured, not that
its workload is available. This `reconcile` plugs into the entrypoints above;
`updateStatus` computes `Ready` after these steps from the observed Deployment.

```go
func (r *FooReconciler) reconcile(ctx context.Context, foo *cachev1.Foo) (ctrl.Result, error) {
    if err := r.ensureDeployment(ctx, foo); err != nil {
        return ctrl.Result{}, err
    }
    if meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
        Type: "DeploymentEnsured", Status: metav1.ConditionTrue,
        Reason: "DeploymentEnsured", Message: "Deployment spec applied; ensuring Service",
        ObservedGeneration: foo.Generation,
    }) {
        if err := r.Status().Update(ctx, foo); err != nil {
            return ctrl.Result{}, err
        }
    }

    if err := r.ensureService(ctx, foo); err != nil {
        return ctrl.Result{}, err
    }
    if meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
        Type: "ResourcesEnsured", Status: metav1.ConditionTrue,
        Reason: "ResourcesEnsured", Message: "Resource specs applied; observing readiness",
        ObservedGeneration: foo.Generation,
    }) {
        if err := r.Status().Update(ctx, foo); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{}, nil
}
```

Batch at the end for fewer API calls; write intermediate progress only when readers
need it. Even with intermediate writes, derive `Ready` from observed state.

### Alternative: MergeFrom patch with explicit ownership

`MergeFrom` is JSON Merge Patch: it replaces changed arrays in full, including
`status.conditions`. The `listType=map` / `listMapKey=type` markers do not change
this behavior. It is not protection against concurrent condition writers; CRDs
also do not support strategic merge patch. See
[controller-runtime v0.23.0 patch implementation](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.0/pkg/client/patch.go).

For a **single status writer**, copy the baseline before changing any status field:

```go
base := foo.DeepCopy()
foo.Status.ObservedGeneration = foo.Generation
changed := meta.SetStatusCondition(&foo.Status.Conditions, condition)
if changed || base.Status.ObservedGeneration != foo.Status.ObservedGeneration {
    if err := r.Status().Patch(ctx, foo, client.MergeFrom(base)); err != nil {
        return ctrl.Result{}, err
    }
}
```

For **multiple writers**, assign each condition type to one owner. All writers must
use concurrency control: read a fresh object, merge only their own conditions into
its current list, and retry conflicts with another read. This example uses an
uncached `reader` (`mgr.GetAPIReader()`) and an optimistic resource-version lock.
The desired condition comes from observations of `observedFoo`; stop and requeue
for fresh reconciliation if its UID or spec generation has changed.

```go
func patchReady(ctx context.Context, c client.Client, reader client.Reader, observedFoo *cachev1.Foo, condition metav1.Condition) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        var current cachev1.Foo
        if err := reader.Get(ctx, client.ObjectKeyFromObject(observedFoo), &current); err != nil {
            return err
        }
        if current.UID != observedFoo.UID || current.Generation != observedFoo.Generation ||
            !current.DeletionTimestamp.IsZero() {
            return fmt.Errorf("Foo changed since readiness was observed; reconcile again")
        }
        base := current.DeepCopy()
        // This writer owns Ready only; preserve every other condition and status field.
        if !meta.SetStatusCondition(&current.Status.Conditions, condition) {
            return nil
        }
        patch := client.MergeFromWithOptions(base, client.MergeFromWithOptimisticLock{})
        return c.Status().Patch(ctx, &current, patch)
    })
}
```

Return `patchReady` errors from reconciliation so exhausted retries or changed
observations trigger another attempt. Do not retry the same stale array. With two
writers starting from `[Ready=False, BackupReady=False]`, a competing BackupReady
update makes the Ready patch conflict; the next read includes `BackupReady=True`,
so the retried list preserves both updates. Never replace the fresh status with the
old snapshot or let both writers own the same condition type. Assign a separate
owner for top-level `observedGeneration` if using it with multiple writers.

[client-go v0.35.0 RetryOnConflict](https://github.com/kubernetes/client-go/blob/v0.35.0/util/retry/util.go)
requires a reread on every attempt. Alternatively, use status SSA with distinct
field managers, the condition map markers above, and only each manager's own
condition entries. Avoid forcing ownership of another writer's fields.

### Required imports

```go
import (
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/util/retry"
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
