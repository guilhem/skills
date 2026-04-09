# Testing Patterns

Patterns for testing Kubebuilder operators with envtest.

## Test Suite Setup

Kubebuilder scaffolds a test suite using Ginkgo + envtest. The suite bootstraps a real API server and etcd without a full cluster.

```go
package controller_test

import (
    "context"
    "path/filepath"
    "testing"
    "time"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/rest"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"

    cachev1 "github.com/example/operator/api/v1alpha1"
)

var (
    cfg       *rest.Config
    k8sClient client.Client
    testEnv   *envtest.Environment
    ctx       context.Context
    cancel    context.CancelFunc
)

func TestControllers(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    ctx, cancel = context.WithCancel(context.TODO())

    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }

    var err error
    cfg, err = testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
    Expect(cfg).NotTo(BeNil())

    err = cachev1.AddToScheme(scheme.Scheme)
    Expect(err).NotTo(HaveOccurred())

    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
    Expect(err).NotTo(HaveOccurred())

    // Start controller manager
    mgr, err := ctrl.NewManager(cfg, ctrl.Options{Scheme: scheme.Scheme})
    Expect(err).NotTo(HaveOccurred())

    err = (&FooReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr)
    Expect(err).NotTo(HaveOccurred())

    go func() {
        defer GinkgoRecover()
        err = mgr.Start(ctx)
        Expect(err).NotTo(HaveOccurred())
    }()
})

var _ = AfterSuite(func() {
    cancel()
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})
```

## Controller Test Pattern

```go
var _ = Describe("Foo Controller", func() {
    const (
        timeout  = 30 * time.Second
        interval = 250 * time.Millisecond
    )

    Context("When creating a Foo resource", func() {
        It("Should create a Deployment", func() {
            foo := &cachev1.Foo{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-foo",
                    Namespace: "default",
                },
                Spec: cachev1.FooSpec{
                    Size: ptr.To(int32(3)),
                },
            }

            // Create the CR
            Expect(k8sClient.Create(ctx, foo)).To(Succeed())

            // Verify Deployment is created
            depKey := types.NamespacedName{Name: "test-foo", Namespace: "default"}
            createdDep := &appsv1.Deployment{}

            Eventually(func() error {
                return k8sClient.Get(ctx, depKey, createdDep)
            }, timeout, interval).Should(Succeed())

            Expect(*createdDep.Spec.Replicas).To(Equal(int32(3)))

            // Verify status is updated
            updatedFoo := &cachev1.Foo{}
            Eventually(func() bool {
                err := k8sClient.Get(ctx, types.NamespacedName{Name: "test-foo", Namespace: "default"}, updatedFoo)
                if err != nil {
                    return false
                }
                return meta.IsStatusConditionTrue(updatedFoo.Status.Conditions, "Ready")
            }, timeout, interval).Should(BeTrue())
        })
    })
})
```

## Testing Webhooks

Enable webhooks in envtest:

```go
testEnv = &envtest.Environment{
    CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
    WebhookInstallOptions: envtest.WebhookInstallOptions{
        Paths: []string{filepath.Join("..", "..", "config", "webhook")},
    },
}
```

Then test validation:

```go
It("Should reject invalid size", func() {
    foo := &cachev1.Foo{
        ObjectMeta: metav1.ObjectMeta{Name: "invalid-foo", Namespace: "default"},
        Spec:       cachev1.FooSpec{Size: ptr.To(int32(100))},
    }
    err := k8sClient.Create(ctx, foo)
    Expect(err).To(HaveOccurred())
    Expect(err.Error()).To(ContainSubstring("must be <= 10"))
})
```

## Running Tests

```bash
# Run all tests
make test

# Run with verbose output
go test ./... -v

# Run specific package
go test ./internal/controller/... -v

# With coverage
go test ./... -coverprofile cover.out
go tool cover -html=cover.out
```

## Test Best Practices

- Use `Eventually` for async assertions (controller reconciliation is asynchronous)
- Set reasonable timeout and interval (30s timeout, 250ms interval is common)
- Clean up resources in `AfterEach` or use unique names per test
- Test both success and error paths
- Test status condition transitions
- Test finalizer behavior (deletion cleanup)
- Use `Consistently` to verify something does NOT happen
- Test idempotency: reconcile should be safe to call multiple times

## Common Pitfalls

- **Missing CRD paths**: `CRDDirectoryPaths` must point to generated CRDs
- **Scheme registration**: register your API types in the test scheme
- **Async assertions**: always use `Eventually`/`Consistently` for controller behavior
- **Test isolation**: each test should create unique resources or clean up properly
- **envtest binaries**: ensure `KUBEBUILDER_ASSETS` points to envtest binaries (usually handled by `setup-envtest`)
