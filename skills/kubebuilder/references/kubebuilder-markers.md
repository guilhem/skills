# Kubebuilder Markers Cheat-Sheet

Use this file when translating API requirements into `+kubebuilder` markers on Go structs.

Canonical references:

- [book.kubebuilder.io/reference/markers.html](https://book.kubebuilder.io/reference/markers.html)
- [book.kubebuilder.io/reference/generating-crd.html](https://book.kubebuilder.io/reference/generating-crd.html)

## Marker Syntax

Controller-gen markers are single-line comments starting with `// +`.

| Form | Example | Usage |
|------|---------|-------|
| Empty (flag) | `// +kubebuilder:validation:Optional` | Boolean toggle |
| Anonymous (single value) | `// +kubebuilder:validation:MaxItems=2` | Single argument |
| Multi-option | `// +kubebuilder:printcolumn:JSONPath=".status.replicas",name=Replicas,type=string` | Named args, comma-separated |

Notes:

- `// +optional` is a Kubernetes codegen convention (often inferred from `omitempty`)
- `// +kubebuilder:validation:Optional` is a controller-gen marker (can be package-wide)
- Value syntax is Go-like (bools, ints, strings). Prefer quoting strings.

## Root Object Markers

Add these above the root type:

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Namespaced,shortName=foo,categories=all
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=`.status.conditions[?(@.type=="Ready")].status`
type Foo struct {
    // ...
}
```

| Marker | Purpose |
|--------|---------|
| `+kubebuilder:object:root=true` | Required for all root types |
| `+kubebuilder:subresource:status` | Enable `/status` subresource (required if you have `.status`) |
| `+kubebuilder:resource:scope=Cluster` | Cluster-scoped (default is Namespaced) |
| `+kubebuilder:resource:shortName=foo` | Short name for kubectl |
| `+kubebuilder:resource:categories=all` | Category grouping |
| `+kubebuilder:subresource:scale:...` | Enable `/scale` subresource (opt-in, needs path mapping) |

## Field-Level Validation

### Strings

```go
// +kubebuilder:validation:MinLength=1
// +kubebuilder:validation:MaxLength=63
// +kubebuilder:validation:Pattern=`^[a-z0-9]([-a-z0-9]*[a-z0-9])?$`
// +kubebuilder:validation:Enum=small;medium;large
Name string `json:"name"`
```

### Numbers

```go
// +kubebuilder:validation:Minimum=0
// +kubebuilder:validation:Maximum=10
// +kubebuilder:validation:MultipleOf=5
// +kubebuilder:validation:ExclusiveMaximum=false
Replicas *int32 `json:"replicas,omitempty"`
```

### Defaults

Defaults must be valid JSON literals. Applied on create only.

```go
// +kubebuilder:default:=3
Replicas *int32 `json:"replicas,omitempty"`

// +kubebuilder:default:="Always"
Policy string `json:"policy,omitempty"`
```

### Objects / Nested Structs (CEL Validation)

```go
// +kubebuilder:validation:XValidation:rule="has(self.foo)",message="foo is required"
Config *FooConfig `json:"config,omitempty"`
```

Note: CEL rules vary by Kubernetes version; use sparingly.

## List and Map Semantics

Critical for SSA (Server-Side Apply) correctness and stable GitOps diffs.

### List item validation

```go
// +kubebuilder:validation:MinItems=1
// +kubebuilder:validation:MaxItems=10
Items []Item `json:"items,omitempty"`
```

### Sets (unique items)

```go
// +kubebuilder:validation:UniqueItems=true
// +kubebuilder:validation:MaxItems=50
// +listType=set
Tags []string `json:"tags,omitempty"`
```

### Map-like lists (keyed by field)

```go
// +listType=map
// +listMapKey=name
Endpoints []Endpoint `json:"endpoints,omitempty"`
```

### Conditions (always map-like, keyed by type)

```go
// +listType=map
// +listMapKey=type
// +kubebuilder:validation:MaxItems=20
// +optional
Conditions []metav1.Condition `json:"conditions,omitempty"`
```

### Atomic lists

```go
// +listType=atomic
Active []corev1.ObjectReference `json:"active,omitempty"`
```

## Printer Columns

Prefer stable JSONPaths. Keep short and user-facing.

```go
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=".status.conditions[?(@.type=='Ready')].status"
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=".metadata.creationTimestamp"
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=".status.phase",priority=0
// +kubebuilder:printcolumn:name="Message",type=string,JSONPath=".status.message",priority=1
```

Higher `priority` columns shown only with `-o wide`.

## Common Pitfalls

- **Markers not appearing in CRD**: confirm they're on the correct field/type and supported by your `controller-gen` version
- **Defaults not applied**: must be valid JSON literals; only applied on create
- **Required vs optional confusion**: requiredness is driven by CRD schema (pointer + omitempty), not by `omitempty` alone
- **List semantics missing**: causes SSA merge conflicts; add `+listType` early
- **Wrong marker scope**: some markers are field-level, some are type-level, some can be package-level
