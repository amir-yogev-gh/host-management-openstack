# host-management-openstack

Kubernetes controller that manages bare metal hosts via OpenStack Ironic. Watches HostLease CRs (defined in bare-metal-fulfillment-operator) and reconciles power state with Ironic nodes.

## Critical Rules

- **Always `make manifests generate`** after modifying CRD-consuming types
- **Always `make lint`** before committing — fix all golangci-lint issues
- **Never edit** `config/crd/` or `zz_generated.deepcopy.go` — these are generated
- Run `make lint test` before committing

## Dev Environment

```bash
make build                     # Build manager binary to bin/manager
make test                      # Unit tests (excludes e2e)
make test-e2e                  # E2E tests with Kind cluster
make lint                      # golangci-lint
make lint-fix                  # Auto-fix lint issues
make fmt                       # go fmt
make vet                       # go vet

make manifests                 # Generate RBAC manifests
make generate                  # Generate DeepCopy methods

make run                       # Run controller locally
make install                   # Install CRDs into cluster
make deploy IMG=<registry>/host-management-openstack:tag
make undeploy

make image-build IMG=<registry>/host-management-openstack:tag
make image-push IMG=<registry>/host-management-openstack:tag

# Pre-commit hooks
pre-commit run --all-files
```

## Repository Structure

```text
host-management-openstack/
├── cmd/
│   └── main.go                # Controller entry point
├── internal/
│   ├── controller/            # HostLease reconciliation logic
│   └── management/            # OpenStack Ironic client
├── config/
│   ├── default/               # Kustomize default overlay
│   ├── manager/               # Manager deployment manifest
│   ├── rbac/                  # Generated RBAC rules
│   └── samples/               # Example HostLease CRs
├── test/
│   ├── e2e/                   # End-to-end tests
│   └── utils/                 # Test utilities
├── hack/                      # Boilerplate and helper scripts
├── Makefile                   # Build, test, lint, deploy targets
├── go.mod                     # Go 1.26, controller-runtime, gophercloud
└── .golangci.yml              # Linter configuration
```

## Architecture

Single controller watching HostLease CRs:

```text
HostLease CR (spec.poweredOn: true/false)
  ↓ (reconcile)
HostLease Controller
  ↓ (OpenStack API)
Ironic Node (power state)
```

- **HostLease types** are defined in `bare-metal-fulfillment-operator/api/v1alpha1/` — this repo consumes them
- **Management client** (`internal/management/`) abstracts OpenStack Ironic interactions via gophercloud
- Controller syncs `spec.poweredOn` with Ironic node power state, requeues on interval for drift detection
- Handles auth reconnection on token expiry

### Key Files

| File | Purpose |
|------|---------|
| `internal/controller/hostlease_controller.go` | Reconciliation logic |
| `internal/controller/hostlease_names.go` | Naming helpers |
| `internal/management/client.go` | Management client interface |
| `internal/management/openstack.go` | OpenStack Ironic implementation |

## CI

GitHub Actions (`.github/workflows/`):
- **build-image.yaml** — runs tests, builds + pushes container image and manifests
- **pre-commit.yaml** — pre-commit hooks + golangci-lint on PRs

## Code Quality

- **golangci-lint** with dupl, errcheck, ginkgolinter, goconst, gocyclo, revive, staticcheck, and more (see `.golangci.yml`)
- **Pre-commit hooks**: trailing-whitespace, yamllint (strict, excludes `config/`), golangci-lint
- **Tests**: Ginkgo v2 + Gomega with envtest
