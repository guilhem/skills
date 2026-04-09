# Go Type Patterns for Kubernetes APIs

Use this file when mapping an API shape into idiomatic Kubernetes Go types in Kubebuilder projects.

For marker syntax (validation/defaulting, list semantics, printer columns), see [kubebuilder-markers.md](./kubebuilder-markers.md).

## Optional vs Required (Go Struct Design)

General rule: represent "optional" as `omitempty` plus a pointer for scalars.

| Intent | Go type | JSON tag | Notes |
|--------|---------|----------|-------|
| Required scalar | `int32`, `string`, `bool` | `json:"name"` | No `omitempty`, non-pointer |
| Optional scalar | `*int32`, `*string`, `*bool` | `json:"name,omitempty"` | Pointer preserves "unset vs zero" |
| Required struct | `FooConfig` | `json:"config"` | Non-pointer |
| Optional struct | `*FooConfig` | `json:"config,omitempty"` | Pointer |

Why pointers for optional: preserves distinction between "unset" and "set to zero/false/empty".

```go
type FooSpec struct {
    // Required - no omitempty, no pointer
    Name string `json:"name"`

    // Optional - pointer + omitempty
    Replicas *int32 `json:"replicas,omitempty"`

    // Optional struct
    Config *FooConfig `json:"config,omitempty"`
}
```

## Time and Durations

- Timestamps: `metav1.Time` (not `time.Time`)
- Durations: `metav1.Duration` (serializes as string like `"5s"`)

```go
import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

type FooStatus struct {
    LastTransitionTime *metav1.Time `json:"lastTransitionTime,omitempty"`
}
```

Canonical background: [book.kubebuilder.io/cronjob-tutorial/api-design.html](https://book.kubebuilder.io/cronjob-tutorial/api-design.html).

## Resource Quantities

Use `resource.Quantity` for CPU/memory/storage-like values. Avoid `float64` for API fields.

```go
import "k8s.io/apimachinery/pkg/api/resource"

type FooSpec struct {
    // e.g. "100m", "1", "1Gi"
    CPU resource.Quantity `json:"cpu"`
}
```

## Int-or-String

Use `intstr.IntOrString` when the API naturally allows either:

```go
import "k8s.io/apimachinery/pkg/util/intstr"

type FooSpec struct {
    Port intstr.IntOrString `json:"port"`
}
```

## Object References

Prefer purpose-built reference types over generic Kubernetes types. Kubernetes upstream discourages new uses of `corev1.LocalObjectReference` and `corev1.ObjectReference`.

### Same-namespace reference (by name)

```go
// ConfigMapRef identifies a ConfigMap in the same namespace.
type ConfigMapRef struct {
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`
}
```

### Cross-namespace reference (name + namespace)

```go
// SecretRef identifies a Secret. If Namespace is omitted, defaults to resource namespace.
type SecretRef struct {
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`

    // +optional
    // +kubebuilder:validation:MinLength=1
    Namespace string `json:"namespace,omitempty"`
}
```

### Reference design checklist

For each relation, explicitly choose and document:

- **Scope**: same-namespace only vs cross-namespace
- **Identity**: name only vs name+namespace
- **Allowed targets**: fixed Kind (recommended) vs one-of kinds
- **Lifecycle behavior**: what happens if target is missing/deleted/recreated
- **Status mirroring**: resolved details go in Status, not Spec

## Enums and Unions

Enums as typed strings with validation marker:

```go
// +kubebuilder:validation:Enum=Allow;Forbid;Replace
type ConcurrencyPolicy string

const (
    AllowConcurrent   ConcurrencyPolicy = "Allow"
    ForbidConcurrent  ConcurrencyPolicy = "Forbid"
    ReplaceConcurrent ConcurrencyPolicy = "Replace"
)
```

OpenAPI unions (`oneOf`) are limited in CRDs. Prefer explicit structs with clear fields over clever unions.

## Numbers

For API compatibility, use:

- `int32` and `int64` for integers
- `resource.Quantity` for decimals
- Never `float32` or `float64`

## Root Object Boilerplate

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type Foo struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitzero"`

    Spec   FooSpec   `json:"spec"`
    Status FooStatus `json:"status,omitzero"`
}

// +kubebuilder:object:root=true
type FooList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitzero"`
    Items           []Foo `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Foo{}, &FooList{})
}
```
