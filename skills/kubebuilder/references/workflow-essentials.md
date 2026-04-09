# Kubebuilder Workflow Essentials (Scaffolding + Generation)

Use when the user asks about Kubebuilder project setup and regeneration steps.

Canonical background: [book.kubebuilder.io/reference/generating-crd.html](https://book.kubebuilder.io/reference/generating-crd.html).

## Clarify First

- Kubebuilder version (or Operator SDK version if wrapper)
- New project or modifying existing?
- Single-group vs multi-group layout
- Namespaced vs cluster-scoped resource(s)

## Project Initialization

```bash
# Create and enter directory
mkdir $GOPATH/my-operator && cd $GOPATH/my-operator

# Initialize project
kubebuilder init --domain example.com --repo github.com/org/my-operator

# Ensure go.mod module path matches --repo
```

## Scaffold an API

```bash
kubebuilder create api --group cache --version v1alpha1 --kind Memcached
```

This creates:

- `api/v1alpha1/memcached_types.go` — CRD type definitions
- `internal/controller/memcached_controller.go` — Controller skeleton
- Updates to `cmd/main.go` for registration

To scaffold only types (no controller): use flags if your Kubebuilder version supports skipping controller generation.

## Edit API Types

Edit `api/<version>/<kind>_types.go`:

- Define Spec/Status structs
- Add markers for validation, defaults, list semantics
- Keep API types stable and compatible; add new fields as optional

## Regenerate Code and Manifests

After **any** API type change:

```bash
make generate    # Creates DeepCopy implementations in api/<version>/zz_generated.deepcopy.go
make manifests   # Generates CRD YAML (config/crd/bases/), RBAC (config/rbac/), webhook configs
```

Both use [controller-gen](https://book.kubebuilder.io/reference/controller-gen.html) with different flags.

## Verify Generated Output

Inspect `config/crd/bases/<group>_<plural>.yaml` and confirm:

- Required fields match expectations
- Defaults appear where intended
- Validation schema matches (patterns, enums, min/max)
- List semantics correct (`x-kubernetes-list-type`, `x-kubernetes-list-map-keys`)
- Status subresource exists when needed
- Printer columns render correctly

## Run Locally

```bash
# Install CRDs into cluster
make install

# Run controller locally against cluster
make run

# Apply a sample CR
kubectl apply -f config/samples/
```

## Build and Deploy

```bash
# Build container image
make docker-build IMG=<registry>/<image>:<tag>

# Push to registry
make docker-push IMG=<registry>/<image>:<tag>

# Deploy to cluster
make deploy IMG=<registry>/<image>:<tag>
```

## Common Pitfalls

- **"Required" vs "optional" in Go**: optional scalars should be pointers; `omitempty` affects JSON output but requiredness is driven by CRD schema
- **Defaults not showing**: must be valid JSON literals in markers; only applied on create
- **Lists missing SSA semantics**: define `listType=set|map` and `listMapKey` early
- **References**: prefer API-owned reference structs over generic Kubernetes types
- **Subresources are opt-in**: `/status` needs `+kubebuilder:subresource:status`
- **controller-gen version mismatch**: if schemas don't match expectations, inspect the Makefile and pinned version

## Generation Issues

- Re-run `make generate` and `make manifests` after any API type change
- If markers don't show up in CRD, confirm they're on the correct field/type and supported by controller-gen version
- If CRDs won't apply, look for structural schema violations (missing `type`, invalid defaults)

## Multi-Version CRDs

If designing multi-version API (`v1alpha1` → `v1beta1` → `v1`):

- Follow [multiversion tutorial](https://book.kubebuilder.io/multiversion-tutorial/tutorial.html)
- Conversion webhooks needed between versions
- Mark storage version with `+kubebuilder:storageversion`
- Keep hub version stable; spoke versions convert to/from hub
