# Controller Patterns

Patterns for writing Kubebuilder reconcilers (controllers).

## Controller Skeleton

```go
package controller

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    cachev1 "github.com/example/operator/api/v1alpha1"
)

type FooReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}
```

## Classic Reconcile Method

> For most CRs, prefer the [Typed Reconciler](#typed-reconciler-reconcileobjectreconciler) below. Use this classic pattern when you need custom fetch logic (unstructured objects, multi-resource lookups, conditional Get).

Follow this pattern: fetch → check deletion → reconcile → update status.

```go
func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // 1. Fetch the resource
    var foo cachev1.Foo
    if err := r.Get(ctx, req.NamespacedName, &foo); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Check if being deleted (finalizer pattern)
    if !foo.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, &foo)
    }

    // 3. Reconcile desired state
    result, err := r.reconcile(ctx, &foo)

    // 4. Always update status (even on error)
    foo.Status.ObservedGeneration = foo.Generation
    if statusErr := r.Status().Update(ctx, &foo); statusErr != nil {
        logger.Error(statusErr, "Failed to update status")
        return ctrl.Result{}, statusErr
    }

    return result, err
}
```

## Typed Reconciler (`reconcile.ObjectReconciler`)

Since controller-runtime v0.17+, prefer `reconcile.ObjectReconciler[T]` to eliminate the common fetch-object boilerplate. The generic interface receives the already-deserialized object (with NotFound automatically handled):

```go
type FooReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// Reconcile receives the deserialized object directly — no need to call Get() or handle NotFound.
func (r *FooReconciler) Reconcile(ctx context.Context, foo *cachev1.Foo) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // foo is already fetched — go straight to business logic
    if !foo.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, foo)
    }

    result, err := r.reconcile(ctx, foo)

    foo.Status.ObservedGeneration = foo.Generation
    if statusErr := r.Status().Update(ctx, foo); statusErr != nil {
        logger.Error(statusErr, "Failed to update status")
        return ctrl.Result{}, statusErr
    }

    return result, err
}
```

Wire it up with `reconcile.AsReconciler`:

```go
func (r *FooReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&cachev1.Foo{}).
        Owns(&appsv1.Deployment{}).
        Complete(reconcile.AsReconciler(mgr.GetClient(), r))
}
```

Key differences from the classic `reconcile.Reconciler`:

- Signature: `Reconcile(ctx, *T)` instead of `Reconcile(ctx, Request)`
- Object fetch + `IgnoreNotFound` is handled by the adapter — no manual `Get()` call
- Use `reconcile.AsReconciler(client, impl)` to convert to a standard `Reconciler`

## SetupWithManager

Configure watches: `For()` the CR, `Owns()` managed resources.

```go
func (r *FooReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&cachev1.Foo{}).
        Owns(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Complete(r)
}
```

`Owns()` watches resources with matching ownerRef, triggering reconciliation of the parent.

## Owner References

Set ownerRef on all resources the controller creates:

```go
if err := ctrl.SetControllerReference(foo, dep, r.Scheme); err != nil {
    return ctrl.Result{}, err
}
```

Benefits:

- Garbage collection: child resources deleted when parent is deleted
- Watch filtering: controller only watches its own resources
- Cascading events: changes to owned resources trigger reconciliation

## Return Options

| Return | Effect |
|--------|--------|
| `ctrl.Result{}, nil` | Success, stop reconciliation |
| `ctrl.Result{}, err` | Error, requeue with backoff |
| `ctrl.Result{Requeue: true}, nil` | Requeue immediately (no error) |
| `ctrl.Result{RequeueAfter: 30*time.Second}, nil` | Requeue after delay |

## Finalizer Pattern

Use finalizers when cleanup is needed before deletion:

```go
const finalizerName = "cache.example.com/finalizer"

func (r *FooReconciler) handleDeletion(ctx context.Context, foo *cachev1.Foo) (ctrl.Result, error) {
    if controllerutil.ContainsFinalizer(foo, finalizerName) {
        // Run cleanup logic
        if err := r.cleanupResources(ctx, foo); err != nil {
            return ctrl.Result{}, err
        }

        // Remove finalizer
        controllerutil.RemoveFinalizer(foo, finalizerName)
        if err := r.Update(ctx, foo); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{}, nil
}

func (r *FooReconciler) ensureFinalizer(ctx context.Context, foo *cachev1.Foo) error {
    if !controllerutil.ContainsFinalizer(foo, finalizerName) {
        controllerutil.AddFinalizer(foo, finalizerName)
        return r.Update(ctx, foo)
    }
    return nil
}
```

## Create-or-Update Pattern

Use `controllerutil.CreateOrUpdate` for idempotent resource management:

```go
dep := &appsv1.Deployment{
    ObjectMeta: metav1.ObjectMeta{
        Name:      foo.Name,
        Namespace: foo.Namespace,
    },
}

if op, err := controllerutil.CreateOrUpdate(ctx, r.Client, dep, func() error {
    // Mutate the deployment to desired state
    dep.Spec.Replicas = foo.Spec.Replicas
    dep.Spec.Template = desiredPodTemplate(foo)
    return ctrl.SetControllerReference(foo, dep, r.Scheme)
}); err != nil {
    r.Recorder.Eventf(foo, corev1.EventTypeWarning, "CreateUpdateFailed",
        "Failed to create or update Deployment %s: %v", dep.Name, err)
    return ctrl.Result{}, err
} else if op != controllerutil.OperationResultNone {
    r.Recorder.Eventf(foo, corev1.EventTypeNormal, string(op),
        "%s Deployment %s", op, dep.Name)
}
```

## Server-Side Apply (SSA) — Native Support

Since controller-runtime **v0.22+** (PR [#3253](https://github.com/kubernetes-sigs/controller-runtime/pull/3253), tracking issue [#3183](https://github.com/kubernetes-sigs/controller-runtime/issues/3183)), the client exposes a first-class `Apply` method that accepts typed `ApplyConfiguration` objects. **Prefer SSA over `CreateOrUpdate` for new controllers** — it is declarative, conflict-safe, and eliminates read-modify-write races.

### Why SSA over CreateOrUpdate

| `controllerutil.CreateOrUpdate` | `client.Apply` (SSA) |
| --- | --- |
| Read-modify-write: `Get` → mutate → `Update` (3 API calls) | Single `Apply` call (1 API call) |
| Susceptible to update conflicts (resource version mismatch) | Conflict-free via field ownership |
| Entire object is sent — accidental field removal if you forget a field | Only declared fields are owned — others left untouched |
| Manual diff logic needed | API server diffs for you |

### Prerequisites

1. **Generate ApplyConfigurations** with `controller-gen` (controller-tools ≥ 0.17):

   ```bash
   # In your Makefile / generate target
   controller-gen applyconfiguration paths="./api/..." output:applyconfiguration:dir=api/v1alpha1/applyconfiguration
   ```

   This produces typed builders (e.g. `FooApplyConfiguration`) under an `applyconfiguration` package.

2. **Import the generated package** alongside your API types.

### Basic Apply Pattern

```go
import (
    ac "github.com/example/operator/api/v1alpha1/applyconfiguration"
    appsac "k8s.io/client-go/applyconfigurations/apps/v1"
    corev1ac "k8s.io/client-go/applyconfigurations/core/v1"
    metav1ac "k8s.io/client-go/applyconfigurations/meta/v1"
)

const fieldManager = "foo-controller"

func (r *FooReconciler) reconcile(ctx context.Context, foo *cachev1.Foo) (ctrl.Result, error) {
    // Build the desired Deployment via ApplyConfiguration
    dep := appsac.Deployment(foo.Name, foo.Namespace).
        WithSpec(appsac.DeploymentSpec().
            WithReplicas(*foo.Spec.Replicas).
            WithSelector(metav1ac.LabelSelector().WithMatchLabels(map[string]string{
                "app": foo.Name,
            })).
            WithTemplate(corev1ac.PodTemplateSpec().
                WithLabels(map[string]string{"app": foo.Name}).
                WithSpec(desiredPodSpec(foo)),
            ),
        )

    // Single declarative call — creates or patches as needed
    if err := r.Apply(ctx, dep, client.FieldOwner(fieldManager), client.ForceOwnership); err != nil {
        return ctrl.Result{}, fmt.Errorf("applying Deployment: %w", err)
    }

    return ctrl.Result{}, nil
}
```

### Status Sub-Resource Apply

Since controller-runtime **v0.23+** (PR [#3321](https://github.com/kubernetes-sigs/controller-runtime/pull/3321)), SSA also works on sub-resources:

```go
statusApply := ac.Foo(foo.Name, foo.Namespace).
    WithStatus(ac.FooStatus().
        WithObservedGeneration(foo.Generation).
        WithConditions(/* ... */),
    )

if err := r.Status().Apply(ctx, statusApply, client.FieldOwner(fieldManager), client.ForceOwnership); err != nil {
    return ctrl.Result{}, fmt.Errorf("applying status: %w", err)
}
```

### Apply Options

| Option | Purpose |
| --- | --- |
| `client.FieldOwner("name")` | **Required** — identifies the field manager |
| `client.ForceOwnership` | Re-acquire conflicting fields owned by other managers (most controllers should use this) |
| `client.DryRunAll` | Validate without persisting |

### Owner References with SSA

Since the `ApplyConfiguration` is not an `runtime.Object`, `ctrl.SetControllerReference` does not work directly. Set the owner ref in the apply configuration:

```go
dep := appsac.Deployment(foo.Name, foo.Namespace).
    WithOwnerReferences(metav1ac.OwnerReference().
        WithAPIVersion(cachev1.GroupVersion.String()).
        WithKind("Foo").
        WithName(foo.Name).
        WithUID(foo.UID).
        WithController(true).
        WithBlockOwnerDeletion(true),
    )
```

### Migration from CreateOrUpdate to SSA

1. Add `controller-gen applyconfiguration` to your code generation pipeline
2. Replace `controllerutil.CreateOrUpdate` blocks with `r.Apply(ctx, applyConfig, ...)`
3. Replace `r.Status().Update` with `r.Status().Apply` where possible
4. Remove manual `Get` + conflict-retry loops — SSA handles this
5. Ensure `FieldOwner` is set consistently across all Apply calls for the same controller

### Legacy SSA via Patch (pre-v0.22)

Before native support, SSA was possible via `Patch` with `client.Apply`. This still works but is **deprecated** in favor of native `Apply`:

```go
// Legacy approach — still works, but prefer client.Apply above
if err := r.Patch(ctx, dep, client.Apply, client.FieldOwner(fieldManager), client.ForceOwnership); err != nil {
    return ctrl.Result{}, err
}
```

## Error Handling Best Practices

- Use `client.IgnoreNotFound(err)` for Get operations (resource may be deleted)
- Always update status even when reconciliation fails
- Use `Requeue: true` for transient conditions (waiting for dependency)
- Use `RequeueAfter` for polling patterns
- Log errors with structured fields: `logger.Error(err, "msg", "key", value)`

## Event Recording for Observability

Events are the primary observability surface for operators — they appear in `kubectl describe`, are collected by monitoring stacks, and give users a timeline of what the controller did and why.

### Setup

Add `record.EventRecorder` to the reconciler struct and register it in `SetupWithManager`:

```go
type FooReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
}

func (r *FooReconciler) SetupWithManager(mgr ctrl.Manager) error {
    r.Recorder = mgr.GetEventRecorderFor("foo-controller")
    return ctrl.NewControllerManagedBy(mgr).
        For(&cachev1.Foo{}).
        Complete(r)
}
```

### Recording Events

Use `Eventf` with the object, event type, reason (PascalCase), and a human-readable message:

```go
// Success events — EventTypeNormal
r.Recorder.Eventf(foo, corev1.EventTypeNormal, "Created", "Created deployment %s", dep.Name)
r.Recorder.Eventf(foo, corev1.EventTypeNormal, "Updated", "Scaled deployment %s to %d replicas", dep.Name, *foo.Spec.Replicas)
r.Recorder.Eventf(foo, corev1.EventTypeNormal, "Deleted", "Cleaned up orphaned ConfigMap %s", cm.Name)

// Failure events — EventTypeWarning
r.Recorder.Eventf(foo, corev1.EventTypeWarning, "CreateFailed", "Failed to create deployment: %v", err)
r.Recorder.Eventf(foo, corev1.EventTypeWarning, "DependencyMissing", "Required Secret %s/%s not found", ns, name)
r.Recorder.Eventf(foo, corev1.EventTypeWarning, "ValidationFailed", "Spec field .replicas exceeds cluster limit")
```

### When to Emit Events

| Situation | Event type | Example reason |
| --- | --- | --- |
| Managed resource created | Normal | `Created` |
| Managed resource updated | Normal | `Updated` |
| Managed resource deleted / cleaned up | Normal | `Deleted`, `CleanedUp` |
| External dependency missing | Warning | `DependencyMissing` |
| Reconciliation failed (transient) | Warning | `RetryableError` |
| Reconciliation failed (permanent) | Warning | `Failed`, `ValidationFailed` |
| Spec change detected | Normal | `SpecChanged` |
| Finalizer added / removed | Normal | `FinalizerAdded`, `FinalizerRemoved` |

### Best Practices

- **PascalCase reasons**: reasons are machine-parseable identifiers — use `CreateFailed`, not `create failed`
- **Stable reason vocabulary**: define reason constants so monitoring alerts can rely on them:
  ```go
  const (
      ReasonCreated           = "Created"
      ReasonCreateFailed      = "CreateFailed"
      ReasonDependencyMissing = "DependencyMissing"
  )
  ```
- **One event per state transition**: don't emit on every reconcile loop — only when something actually changes
- **Avoid sensitive data**: never include secrets, tokens, or PII in event messages
- **Pair with conditions**: events give the timeline, conditions give current state — emit both. Always set `ObservedGeneration` on conditions and use `SetStatusCondition`'s bool return to skip unnecessary status writes *and* duplicate events (see [status and conditions](./status-and-conditions.md)):
  ```go
  condition := metav1.Condition{
      Type:               "Ready",
      Status:             metav1.ConditionFalse,
      Reason:             ReasonDependencyMissing,
      Message:            fmt.Sprintf("Secret %s/%s not found", ns, name),
      ObservedGeneration: foo.Generation,
  }
  if meta.SetStatusCondition(&foo.Status.Conditions, condition) {
      // State actually changed — persist and emit event
      if err := r.Status().Update(ctx, foo); err != nil {
          return ctrl.Result{}, err
      }
      r.Recorder.Eventf(foo, corev1.EventTypeWarning, ReasonDependencyMissing,
          "Required Secret %s/%s not found", ns, name)
  }
  ```
  Placing the event inside `if SetStatusCondition(...)` enforces "one event per state transition" — no duplicate Warning events on every requeue while the dependency is still missing.
- **RBAC**: the controller needs `events.k8s.io` permissions — see [RBAC markers](./rbac-markers.md#for-events):
  ```go
  // +kubebuilder:rbac:groups=events.k8s.io,resources=events,verbs=create;patch
  // +kubebuilder:rbac:groups="",resources=events,verbs=create;patch
  ```
  Both the modern (`events.k8s.io`) and legacy (`""`) groups are needed for full compatibility.
