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
5. **References**: target kind(s), same-namespace by default, explicit target authorization if cross-namespace, required?, behavior when target missing. See [reference authorization](./references/go-type-patterns.md#reference-authorization).

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

**Integrate** — reuse the project's lint target, configuration, custom binary name,
and pinned Go/golangci-lint/KAL versions. Do not replace existing configuration or
upgrade tools just to add KAL. If setup is missing, KAL ships as a `golangci-lint`
v2 module; merge an entry like this into `.custom-gcl.yml`, replacing the version
placeholders with compatible explicit pins:

```yaml
version: "<project-golangci-lint-version>"
name: golangci-lint-kube-api-linter
destination: ./bin
plugins:
  - module: "sigs.k8s.io/kube-api-linter"
    version: "<pinned-KAL-pseudo-version>"
```

Build with the pinned golangci-lint and run **report-only first**, using the
project's configuration and actual API package paths (including multi-group layouts):

```bash
golangci-lint custom          # builds the custom binary
./bin/golangci-lint-kube-api-linter run ./api/... --fix=false
```

**Key linters for CRDs** (upstream defaults at
[KAL `79b557aa8c8d`](https://github.com/kubernetes-sigs/kube-api-linter/blob/79b557aa8c8dc08e801a1d3e6109881e3551bf30/docs/linters.md);
the project's pinned version and configuration determine what runs):

| Linter                      | What it checks                              |
| --------------------------- | ------------------------------------------- |
| `optionalorrequired`        | Every field has `+optional` or `+required`  |
| `optionalfields`            | Optional → pointer + `omitempty`/`omitzero` |
| `requiredfields`            | Required → `omitempty` tag correctness      |
| `conditions`                | `[]metav1.Condition` tags & markers         |
| `ssatags`                   | `listType` / `listMapKey` for SSA           |
| `jsontags`                  | camelCase JSON tags                         |
| `nobools` _(off; opt-in)_    | Flag booleans for a manual API design choice |
| `nophase`                   | No `Phase` fields in status                 |
| `integers`                  | Only `int32` / `int64`                      |
| `maxlength` _(off)_         | Enforce `MaxLength` / `MaxItems` bounds     |
| `statussubresource` _(off)_ | Status subresource marker present           |

**Merge KAL settings and API path exclusions** into the existing `.golangci.yml`;
retain other linters and adapt the path expression to the project:

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

Run KAL after `make generate` / `make manifests`. Triage findings against Step 0:

- For a new, unpublished API, use findings to shape the design according to the
  user's chosen policy. `nobools` is opt-in; boolean-to-enum and Phase redesigns
  are manual decisions, not automatic fixes.
- For a published API or an existing CRD contract, assess Go consumer changes,
  JSON presence/names (`omitempty`/`omitzero`), and generated CRD schema changes
  before adopting pointer or tag fixes. Follow the user's compatibility decision;
  if preserving the contract complicates the change and no policy is specified,
  ask with the maintenance cost and break consequences. Do not silently redesign it.
- Apply `--fix` only after choosing which available mechanical fixes fit that
  scope and policy; narrow the enabled linters/packages accordingly. Review the
  diff, regenerate, and test serialization and schema behavior as appropriate.
  Stylistic findings need not delay writing tests.

### Step 10 — Test

Write controller tests using envtest. See [testing patterns](./references/testing-patterns.md).

### Step 11 — Keep Up to Date (Migrations)

Kubebuilder evolves across releases. Keep the scaffold aligned with ecosystem changes.

| Method                             | Best for                                 |
| ---------------------------------- | ---------------------------------------- |
| AutoUpdate plugin (GitHub Actions) | Ongoing automated upgrades               |
| `kubebuilder alpha update`         | One-off upgrades in an isolated clone    |
| `kubebuilder alpha generate`       | Separate scaffold for a manual merge     |
| Manual migration                   | Pre-v3 projects or non-standard layouts  |

Both alpha commands can clean their working directory, including ignored files;
a clean Git status is not a backup. Use a separate output or disposable clone as
shown in the [migrations reference](./references/migrations.md).

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
