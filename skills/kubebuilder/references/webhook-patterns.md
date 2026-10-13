# Webhook Patterns

Patterns for admission and conversion webhooks in Kubebuilder.

## Scaffold Webhooks

```bash
# Defaulting and validating webhook
kubebuilder create webhook --group cache --version v1alpha1 --kind Memcached \
    --defaulting --programmatic-validation

# Conversion webhook (for multi-version CRDs)
kubebuilder create webhook --group cache --version v1alpha1 --kind Memcached \
    --conversion
```

This creates `api/<version>/<kind>_webhook.go`.

## Defaulting Webhook

Implements `webhook.CustomDefaulter` to set default values:

```go
// +kubebuilder:webhook:path=/mutate-cache-v1alpha1-memcached,mutating=true,failurePolicy=fail,sideEffects=None,groups=cache.example.com,resources=memcacheds,verbs=create;update,versions=v1alpha1,name=mmemcached.kb.io,admissionReviewVersions=v1

type MemcachedCustomDefaulter struct{}

var _ webhook.CustomDefaulter = &MemcachedCustomDefaulter{}

func (d *MemcachedCustomDefaulter) Default(ctx context.Context, obj runtime.Object) error {
    memcached, ok := obj.(*cachev1.Memcached)
    if !ok {
        return fmt.Errorf("expected Memcached, got %T", obj)
    }

    // Set defaults
    if memcached.Spec.Size == nil {
        defaultSize := int32(1)
        memcached.Spec.Size = &defaultSize
    }

    return nil
}
```

## Validating Webhook

Implements `webhook.CustomValidator` to reject invalid resources:

```go
// +kubebuilder:webhook:path=/validate-cache-v1alpha1-memcached,mutating=false,failurePolicy=fail,sideEffects=None,groups=cache.example.com,resources=memcacheds,verbs=create;update;delete,versions=v1alpha1,name=vmemcached.kb.io,admissionReviewVersions=v1

type MemcachedCustomValidator struct{}

var _ webhook.CustomValidator = &MemcachedCustomValidator{}

func (v *MemcachedCustomValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    memcached, ok := obj.(*cachev1.Memcached)
    if !ok {
        return nil, fmt.Errorf("expected Memcached, got %T", obj)
    }
    return v.validate(memcached)
}

func (v *MemcachedCustomValidator) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) (admission.Warnings, error) {
    newMemcached, ok := newObj.(*cachev1.Memcached)
    if !ok {
        return nil, fmt.Errorf("expected Memcached, got %T", newObj)
    }
    oldMemcached, ok := oldObj.(*cachev1.Memcached)
    if !ok {
        return nil, fmt.Errorf("expected Memcached, got %T", oldObj)
    }

    // Validate immutable fields
    if newMemcached.Spec.StorageClass != oldMemcached.Spec.StorageClass {
        return nil, field.Forbidden(
            field.NewPath("spec", "storageClass"),
            "storageClass is immutable after creation",
        )
    }

    return v.validate(newMemcached)
}

func (v *MemcachedCustomValidator) ValidateDelete(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    return nil, nil
}

func (v *MemcachedCustomValidator) validate(m *cachev1.Memcached) (admission.Warnings, error) {
    var allErrs field.ErrorList

    if m.Spec.Size != nil && *m.Spec.Size > 10 {
        allErrs = append(allErrs, field.Invalid(
            field.NewPath("spec", "size"),
            *m.Spec.Size,
            "must be <= 10",
        ))
    }

    if len(allErrs) > 0 {
        return nil, apierrors.NewInvalid(
            schema.GroupKind{Group: "cache.example.com", Kind: "Memcached"},
            m.Name,
            allErrs,
        )
    }
    return nil, nil
}
```

## Register Webhooks in main.go

```go
if err = ctrl.NewWebhookManagedBy(mgr, &cachev1.Memcached{}).
    WithCustomDefaulter(&MemcachedCustomDefaulter{}).
    WithCustomValidator(&MemcachedCustomValidator{}).
    Complete(); err != nil {
    setupLog.Error(err, "unable to create webhook", "webhook", "Memcached")
    os.Exit(1)
}
```

## Conversion Webhooks (Multi-Version)

When versions need field conversion, implement hub-and-spoke conversion. Identical
schemas can use `spec.conversion.strategy: None`, which only changes `apiVersion`.

```go
// Hub version (storage version)
func (m *Memcached) Hub() {}

// Spoke version converts to/from hub
func (m *MemcachedV1beta1) ConvertTo(dstRaw conversion.Hub) error {
    dst := dstRaw.(*Memcached)
    dst.ObjectMeta = *m.ObjectMeta.DeepCopy()
    // Convert fields from v1beta1 to v1
    dst.Spec.Size = m.Spec.Replicas  // field renamed
    return nil
}

func (m *MemcachedV1beta1) ConvertFrom(srcRaw conversion.Hub) error {
    src := srcRaw.(*Memcached)
    m.ObjectMeta = *src.ObjectMeta.DeepCopy()
    // Convert fields from v1 to v1beta1
    m.Spec.Replicas = src.Spec.Size
    return nil
}
```

This abbreviated example maps only `Replicas` ↔ `Size`. Real conversions must
map all relevant spec and status fields, including a preservation strategy for
fields absent from another version. Test round trips in both directions for the
mapped value and metadata (name, namespace, UID, labels, annotations, owner
references, and finalizers); converted metadata must not alias the source.

[Controller-runtime v0.23.0 allocates the destination](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.0/pkg/webhook/conversion/conversion.go)
before conversion. Preserve `ObjectMeta` and leave the destination `TypeMeta`
alone: the handler sets the requested API version and kind. Follow the
[storage migration workflow](./workflow-essentials.md#multi-version-crds) before
retiring a version.

## Immutable Fields with CEL

Prefer CRD CEL transition rules for schema-expressible constraints on supported
clusters. [Validation rules are stable in Kubernetes v1.29](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation-rules);
they were [beta and enabled by default in v1.25](https://kubernetes.io/blog/2022/09/23/crd-validation-rules-beta/)
and [alpha in v1.23](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates-removed/).
Check the target cluster's feature gate and generator support
when using earlier releases. Use `ValidateUpdate` for constraints beyond available
schema/CEL capabilities, such as checks involving another object.

For a required, non-nullable field, combine schema requiredness with a value rule:

```go
type MemcachedSpec struct {
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MaxLength=253
    // +kubebuilder:validation:XValidation:rule="self == oldSelf",message="storageClass is immutable"
    StorageClass string `json:"storageClass"`
}
```

For an optional, non-nullable field whose presence and value must be fixed at
creation, put both rules on the required parent object:

```go
// +kubebuilder:validation:XValidation:rule="has(self.storageClass) == has(oldSelf.storageClass)",message="storageClass presence is immutable"
// +kubebuilder:validation:XValidation:rule="!has(self.storageClass) || !has(oldSelf.storageClass) || self.storageClass == oldSelf.storageClass",message="storageClass is immutable"
type MemcachedSpec struct {
    // +kubebuilder:validation:Optional
    // +kubebuilder:validation:MaxLength=253
    StorageClass *string `json:"storageClass,omitempty"`
}
```

In both examples, mark the root object's `Spec MemcachedSpec` field `Required`
and use `json:"spec"` so updates cannot bypass the rules by removing `spec`.
The optional example allows creation with or without `storageClass`; updates
allow absent → absent and equal present values, but reject addition, removal,
and value changes. Defaults are applied before validation, so a defaulted value
counts as present. [Field-level transition rules](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#transition-rules)
run only when both values exist; parent rules catch presence changes. These
examples need no `optionalOldSelf` support. Validate the generated CRD and these
transitions against the target API server; generation alone does not prove acceptance.

## Webhook Configuration

After scaffolding, verify:

- `config/webhook/` contains the webhook manifests
- `config/certmanager/` has cert-manager resources (webhooks need TLS)
- `config/default/kustomization.yaml` enables webhook and certmanager patches

## Testing Webhooks Locally

- Webhooks require TLS certificates
- For local development, use `make run` with webhook disabled, or
- Use `kind` cluster with cert-manager installed
- Run `make install` to install CRDs, then `make run` for the controller

## Common Pitfalls

- **Missing cert-manager**: webhooks won't work without TLS; install cert-manager first
- **Webhook path mismatch**: marker path must match the registration in main.go
- **sideEffects**: always set `sideEffects=None` unless the webhook truly has side effects
- **failurePolicy**: use `Fail` in production, `Ignore` only during development
- **Immutable field validation**: use CEL where supported; optional fields need explicit presence rules (see above)
