---
name: kubebuilder
description: "Build Kubernetes operators with Kubebuilder in Go: project scaffolding, API type design, CRD markers, controllers, webhooks, RBAC, testing, and the generate/manifest workflow. Use when creating or modifying Kubebuilder projects, designing CRD APIs, writing reconcilers, adding admission webhooks, or troubleshooting operator scaffolding and generation."
---

# Kubebuilder (Go Kubernetes Operators)

Build Kubernetes operators with Kubebuilder: project init, API design (Go types + markers), controllers, webhooks, RBAC, testing, and the full scaffold/generate/deploy workflow.

## When to Use

- Initializing a new Kubebuilder operator project
- Designing or modifying CRD API types (Spec, Status, markers)
- Writing or debugging reconciler (controller) logic
- Adding admission or conversion webhooks
- Configuring RBAC markers and permissions
- Running `make generate` / `make manifests` and troubleshooting output
- Writing controller tests with envtest
- Multi-version CRD design and conversion

## Tooling Assumptions

- `go` toolchain (matching `go.mod`)
- `kubebuilder` CLI (or Operator SDK wrapper)
- `make` (Kubebuilder projects use Makefile for generation)
- `controller-gen` installed automatically by Makefile targets
- Cluster tools (`kubectl`, `kind`) only needed for running/testing

## Workflow

### Step 0 — Gather Inputs

Determine before writing any code:

1. **API identity**: Group, Version, Kind, Plural, Scope (Namespaced vs Cluster)
2. **Spec fields**: required vs optional, immutability expectations
3. **Status needs**: conditions? observedGeneration? summary fields?
4. **Constraints**: enum values, ranges, patterns, max items, uniqueness
5. **References**: target kind(s), cross-namespace?, required?, behavior when target missing

If the user already has CRD YAML, treat it as the contract and map into Go types + markers.

### Step 1 — Scaffold Project (New Projects Only)

```bash
kubebuilder init --domain <domain> --repo <go-module>
kubebuilder create api --group <group> --version <version> --kind <Kind>
```

Clarify: single-group vs multi-group layout, Kubebuilder version. See [workflow essentials](./references/workflow-essentials.md).

### Step 2 — Design Go API Types

Create `api/<version>/<kind>_types.go` with:

- `type <Kind>Spec struct { ... }`
- `type <Kind>Status struct { ... }`
- `type <Kind> struct { metav1.TypeMeta; metav1.ObjectMeta; Spec; Status }`
- `type <Kind>List struct { metav1.TypeMeta; metav1.ListMeta; Items []<Kind> }`

Rules:

- Use pointers for optional scalars/structs to preserve "unset vs zero"
- Prefer Kubernetes-native types: `metav1.Time`, `resource.Quantity`, `intstr.IntOrString`
- Keep Status separate from Spec
- For references, define purpose-built `<Thing>Ref` structs (avoid generic `corev1.ObjectReference`)

See [Go type patterns](./references/go-type-patterns.md).

### Step 3 — Add Kubebuilder Markers

Add markers for validation, defaults, list semantics, and printer columns:

- Root: `+kubebuilder:object:root=true`, `+kubebuilder:subresource:status`
- Fields: validation (min/max, pattern, enum, length), defaults
- Lists: `+listType=set|map`, `+listMapKey=...` for SSA correctness
- Printer columns for `kubectl get` UX

See [Kubebuilder markers reference](./references/kubebuilder-markers.md).

### Step 4 — Shape Status + Conditions

Prefer `[]metav1.Condition` with map-like list semantics (keyed by `type`). Include `ObservedGeneration`. See [status and conditions](./references/status-and-conditions.md).

### Step 5 — Write Controller Logic

Implement the reconciler in `internal/controller/<kind>_controller.go`:

- **Prefer `reconcile.ObjectReconciler[T]`** (controller-runtime v0.17+) to eliminate fetch boilerplate — receives the deserialized object directly, wired via `reconcile.AsReconciler(client, impl)`
- Fall back to classic `reconcile.Reconciler` when custom fetch logic is needed (unstructured, multi-object)
- Set up watches: `For()` the CR, `Owns()` managed resources
- Use `ctrl.SetControllerReference()` for owned resources
- Handle create/update/delete with proper error handling and requeueing
- Update status conditions after each reconciliation
- Record Kubernetes Events at each meaningful state transition for observability

**Events for observability**: emit `Normal` events on success (resource created/updated/deleted) and `Warning` events on failures or missing dependencies. Use stable PascalCase reason constants. Pair events with condition updates — events give the timeline, conditions give current state. See [event recording for observability](./references/controller-patterns.md#event-recording-for-observability).

See [controller patterns](./references/controller-patterns.md).

### Step 6 — Add RBAC Markers

Add RBAC markers above the `Reconcile()` method. Run `make manifests` to regenerate `config/rbac/`. See [RBAC markers](./references/rbac-markers.md).

### Step 7 — Add Webhooks (If Needed)

Scaffold with `kubebuilder create webhook`. Implement defaulting and/or validating logic. See [webhook patterns](./references/webhook-patterns.md).

### Step 8 — Generate and Verify

```bash
make generate    # DeepCopy implementations
make manifests   # CRD YAML, RBAC, webhook configs
```

Inspect `config/crd/bases/` and confirm: required fields, defaults, list semantics, printer columns, status subresource.

### Step 9 — Lint API Types with KAL

[Kube API Linter (KAL)](https://github.com/kubernetes-sigs/kube-api-linter) is an official `kubernetes-sigs` linter that enforces the [K8s API Conventions](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md) on Go types. It catches mechanical API review issues before human review.

**Install** — KAL ships as a `golangci-lint` v2 module. Add a `.custom-gcl.yml`:

```yaml
version: v2.5.0
name: golangci-lint-kube-api-linter
destination: ./bin
plugins:
  - module: "sigs.k8s.io/kube-api-linter"
    version: "<pseudo-version>" # check pkg.go.dev for latest
```

Then build and run:

```bash
golangci-lint custom          # builds the custom binary
./bin/golangci-lint-kube-api-linter run ./api/...  --fix
```

**Key linters for CRDs** (enabled by default unless noted):

| Linter                      | What it checks                              |
| --------------------------- | ------------------------------------------- |
| `optionalorrequired`        | Every field has `+optional` or `+required`  |
| `optionalfields`            | Optional → pointer + `omitempty`/`omitzero` |
| `requiredfields`            | Required → `omitempty` tag correctness      |
| `conditions`                | `[]metav1.Condition` tags & markers         |
| `ssatags`                   | `listType` / `listMapKey` for SSA           |
| `jsontags`                  | camelCase JSON tags                         |
| `nobools`                   | Avoid bare `bool` (use string enums)        |
| `nophase`                   | No `Phase` fields in status                 |
| `integers`                  | Only `int32` / `int64`                      |
| `maxlength` _(off)_         | Enforce `MaxLength` / `MaxItems` bounds     |
| `statussubresource` _(off)_ | Status subresource marker present           |

**Restrict to API paths** in `.golangci.yml`:

```yaml
version: "2"
linters:
  enable:
    - kubeapilinter
  settings:
    custom:
      kubeapilinter:
        type: module
        settings:
          linters: {}
          lintersConfig: {}
  exclusions:
    rules:
      - linters: [kubeapilinter]
        path-except: api/*
```

Run KAL after `make generate` / `make manifests` and fix findings before writing tests.

### Step 10 — Test

Write controller tests using envtest. See [testing patterns](./references/testing-patterns.md).

### Step 11 — Keep Up to Date (Migrations)

Kubebuilder evolves across releases. Keep the scaffold aligned with ecosystem changes.

| Method                             | Best for                                 |
| ---------------------------------- | ---------------------------------------- |
| AutoUpdate plugin (GitHub Actions) | Ongoing automated upgrades               |
| `kubebuilder alpha update`         | Local one-off upgrades                   |
| `kubebuilder alpha generate`       | Baseline for heavily customized projects |
| Manual migration                   | Pre-v3 projects or non-standard layouts  |

See [migrations reference](./references/migrations.md).

## Common Pitfalls

- **Operator not at git root** (monorepo): CLI, GitHub Action, and Makefile all
  assume the operator is at the working directory root. See [migrations pitfalls](./references/migrations.md#operator-not-at-git-root-monorepo).
- **Scaffold files moved or renamed**: breaks automated migration tools. Keep the
  standard Kubebuilder layout intact.
- **Manually created APIs not tracked**: resources not scaffolded via CLI won't
  appear in `PROJECT` and won't be handled by migration tools.

## Output Format

When delivering code, output a single Go snippet per file:

1. Package + imports
2. Markers
3. Spec, Status
4. Root object + list
5. Type aliases and nested structs

Use concise comments. Put rationale (why a pointer, why listType map, etc.) in short bullet notes after the code.

## Resources

- [Kubebuilder Book](https://book.kubebuilder.io/)
- [Markers reference](https://book.kubebuilder.io/reference/markers.html)
- [CRD generation](https://book.kubebuilder.io/reference/generating-crd.html)
- [API design tutorial](https://book.kubebuilder.io/cronjob-tutorial/api-design.html)
- [K8s API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [Multiversion tutorial](https://book.kubebuilder.io/multiversion-tutorial/tutorial.html)
- [Kube API Linter (KAL)](https://github.com/kubernetes-sigs/kube-api-linter) — lint Go API types against K8s API conventions
- [KAL linters reference](https://github.com/kubernetes-sigs/kube-api-linter/blob/main/docs/linters.md)

### Reference Files

Load as needed (keep context small):

- [Go type patterns](./references/go-type-patterns.md)
- [Kubebuilder markers](./references/kubebuilder-markers.md)
- [Status and conditions](./references/status-and-conditions.md)
- [Workflow essentials](./references/workflow-essentials.md)
- [Controller patterns](./references/controller-patterns.md)
- [Webhook patterns](./references/webhook-patterns.md)
- [RBAC markers](./references/rbac-markers.md)
- [Testing patterns](./references/testing-patterns.md)
- [Migrations](./references/migrations.md)
