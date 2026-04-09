# RBAC Markers

RBAC markers generate ClusterRole rules in `config/rbac/role.yaml`.

## Marker Syntax

Place RBAC markers above the `Reconcile()` method:

```go
// +kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=events.k8s.io,resources=events,verbs=create;patch
func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
```

## Common RBAC Patterns

### For the Custom Resource itself

```go
// +kubebuilder:rbac:groups=<group>,resources=<plural>,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=<group>,resources=<plural>/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=<group>,resources=<plural>/finalizers,verbs=update
```

### For managed Deployments

```go
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
```

### For managed Services

```go
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
```

### For ConfigMaps and Secrets (read-only)

```go
// +kubebuilder:rbac:groups=core,resources=configmaps,verbs=get;list;watch
// +kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch
```

### For Events

```go
// +kubebuilder:rbac:groups=events.k8s.io,resources=events,verbs=create;patch
```

### For leader election

Leader election RBAC is typically handled by the manager, not individual controllers. Check `config/rbac/leader_election_role.yaml`.

## Namespace-Scoped vs Cluster-Scoped

- Default: markers generate `ClusterRole` (cluster-wide permissions)
- For namespace-scoped operators, the generated `ClusterRole` is still used but bound via `RoleBinding` to specific namespaces

## Regenerate

After changing RBAC markers:

```bash
make manifests
```

Inspect `config/rbac/role.yaml` to verify rules.

## Principle of Least Privilege

- Only request verbs you actually need
- Use `get;list;watch` for read-only access
- Add `create;update;patch;delete` only for resources the controller manages
- Separate read-only and write RBAC markers for clarity
- Use `resources/<subresource>` for fine-grained access (e.g., `pods/log`, `deployments/scale`)

## Common Pitfalls

- **Missing status RBAC**: without `resources=<plural>/status,verbs=get;update;patch`, status updates fail with 403
- **Missing finalizer RBAC**: without `resources=<plural>/finalizers,verbs=update`, adding finalizers fails
- **Core group**: use `groups=core` (not `groups=""`) for core API resources (pods, services, configmaps)
- **Events group**: use `groups=events.k8s.io` for the modern events API
- **Forgotten `make manifests`**: RBAC changes only take effect after regeneration
