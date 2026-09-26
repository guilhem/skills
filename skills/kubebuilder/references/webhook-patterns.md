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

For multi-version CRDs, implement hub-and-spoke conversion:

```go
// Hub version (storage version)
func (m *Memcached) Hub() {}

// Spoke version converts to/from hub
func (m *MemcachedV1beta1) ConvertTo(dstRaw conversion.Hub) error {
    dst := dstRaw.(*Memcached)
    // Convert fields from v1beta1 to v1
    dst.Spec.Size = m.Spec.Replicas  // field renamed
    return nil
}

func (m *MemcachedV1beta1) ConvertFrom(srcRaw conversion.Hub) error {
    src := srcRaw.(*Memcached)
    // Convert fields from v1 to v1beta1
    m.Spec.Replicas = src.Spec.Size
    return nil
}
```

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
- **Immutable field validation**: must be done in `ValidateUpdate`, not via markers alone
