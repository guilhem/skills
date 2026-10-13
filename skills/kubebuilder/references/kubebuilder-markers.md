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

Defaults must be valid JSON literals. [Schema defaulting](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#defaulting)
uses the request version on API requests and the storage version on reads from
etcd; it also runs after mutating admission patches. A newly added default can
therefore appear on existing objects without recreation. Read-time defaults are
not persisted automatically; an API update is needed to write them to etcd.

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

For required and optional immutability, version support, and presence rules, see
[immutable fields with CEL](./webhook-patterns.md#immutable-fields-with-cel).

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
// +kubebuilder:validation:MaxItems=50
// +listType=set
Tags []string `json:"tags,omitempty"`
```

`+listType=set` generates `x-kubernetes-list-type: set`, which enforces uniqueness
and gives the list set semantics for SSA. Do not add `UniqueItems=true`:
[Kubernetes v1.34.0 CRD validation rejects `uniqueItems: true`](https://github.com/kubernetes/kubernetes/blob/v1.34.0/staging/src/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/validation/validation.go).
Check CRD acceptance and duplicate rejection with API-server validation; successful
generation alone does not establish either.

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
- **Defaults not applied**: check the relevant version's schema; request/read defaults do not imply a stored value (see above)
- **Required vs optional confusion**: requiredness is driven by CRD schema (pointer + omitempty), not by `omitempty` alone
- **List semantics missing**: causes SSA merge conflicts; add `+listType` early
- **Wrong marker scope**: some markers are field-level, some are type-level, some can be package-level
