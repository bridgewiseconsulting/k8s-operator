# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is a Kubernetes operator built with Kubebuilder v4.9.0 that manages `Application` custom resources in the `app.arvin.com` API group. The operator follows standard Kubernetes operator patterns using controller-runtime.

## Development Commands

### Building and Running

```bash
# Build the operator binary
make build

# Run the operator locally (against your current kubeconfig context)
make run

# Build and push Docker image (set IMG variable)
make docker-build docker-push IMG=<registry>/k8s-operator:tag
```

### Code Generation

```bash
# Generate CRD manifests, RBAC, and webhook configurations
make manifests

# Generate deepcopy implementations for API types
make generate

# Run both manifests and generate (recommended after API changes)
make manifests generate
```

### Testing

```bash
# Run unit tests (includes fmt, vet, and envtest setup)
make test

# Run e2e tests (creates a Kind cluster automatically)
make test-e2e

# Run a single test package
go test ./internal/controller/... -v

# Run specific test by name
go test ./internal/controller/... -run TestApplicationController -v
```

### Linting and Formatting

```bash
# Run golangci-lint
make lint

# Run golangci-lint with auto-fix
make lint-fix

# Format code with gofmt
make fmt

# Run go vet
make vet
```

### Cluster Operations

```bash
# Install CRDs into the cluster
make install

# Uninstall CRDs from the cluster
make uninstall

# Deploy the operator to the cluster
make deploy IMG=<registry>/k8s-operator:tag

# Remove the operator from the cluster
make undeploy

# Apply sample Application CR
kubectl apply -k config/samples/

# Delete sample resources
kubectl delete -k config/samples/
```

### Building Distribution Artifacts

```bash
# Generate consolidated install.yaml in dist/
make build-installer IMG=<registry>/k8s-operator:tag
```

## Architecture

### Project Structure

- **`api/v1alpha1/`**: API type definitions for the `Application` CRD
  - `application_types.go`: Defines ApplicationSpec and ApplicationStatus
  - API group: `app.arvin.com`, version: `v1alpha1`
  
- **`internal/controller/`**: Reconciliation logic
  - `application_controller.go`: Core reconciler implementing the control loop
  - Controller watches Application resources and reconciles cluster state
  
- **`cmd/main.go`**: Operator entrypoint
  - Sets up controller-runtime manager with metrics, health checks, and leader election
  - Configures TLS for metrics and webhook servers
  - Supports both HTTP and HTTPS metrics endpoints
  
- **`config/`**: Kustomize manifests for deployment
  - `crd/`: CustomResourceDefinition manifests
  - `rbac/`: ServiceAccount, Role, and RoleBinding definitions
  - `manager/`: Operator Deployment manifest
  - `default/`: Default kustomization overlays
  - `samples/`: Example Application CRs
  - `prometheus/`: Prometheus ServiceMonitor configuration
  - `network-policy/`: NetworkPolicy for metrics access

### Key Dependencies

- **Kubebuilder**: v4.9.0 - Scaffolding framework
- **controller-runtime**: v0.22.1 - Core reconciliation framework
- **Go**: 1.24.5
- **Kubernetes**: Minimum v1.11.3 client, tested against v0.34.0 APIs

### Reconciliation Pattern

The ApplicationReconciler follows the standard Kubernetes controller pattern:
1. Watch for Application resource changes
2. Reconcile function receives requests with namespace/name
3. Fetch current state, compare with desired state
4. Update cluster resources to match desired state
5. Update Application status conditions to reflect observed state

### Code Generation Workflow

When modifying API types (`api/v1alpha1/application_types.go`):
1. Make changes to Spec or Status structs
2. Add kubebuilder markers for validation/defaulting as needed
3. Run `make manifests generate` to update:
   - CRD YAML in `config/crd/bases/`
   - DeepCopy methods in `zz_generated.deepcopy.go`
   - RBAC manifests based on controller markers

### Testing Infrastructure

- **Unit tests**: Use envtest (provides real Kubernetes API server)
- **E2E tests**: Use Kind for isolated cluster testing
- **Test utilities**: Located in `test/utils/`
- Kind cluster name for e2e: `k8s-operator-test-e2e`

## Configuration Notes

### Metrics and Monitoring

- Metrics endpoint defaults to port 8443 (HTTPS) or 8080 (HTTP)
- Supports both secure (with RBAC auth) and insecure modes
- Health probes on `:8081` (`/healthz`, `/readyz`)
- Prometheus ServiceMonitor available in `config/prometheus/`

### Leader Election

- Disabled by default in local development
- Leader election ID: `fe78fabc.arvin.com`
- Enable with `--leader-elect` flag for production deployments

### Linting Configuration

golangci-lint is configured in `.golangci.yml` with:
- Enabled linters: copyloopvar, dupl, errcheck, ginkgolinter, goconst, gocyclo, govet, ineffassign, lll, misspell, nakedret, prealloc, revive, staticcheck, unconvert, unparam, unused
- Auto-formatting with gofmt and goimports
- Exclusions for generated code and API types
