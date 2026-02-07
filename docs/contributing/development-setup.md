# Development Setup

This guide covers setting up a development environment for Butler components.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Repository Setup](#repository-setup)
- [Local Development](#local-development)
- [Testing](#testing)
- [Debugging](#debugging)

---

## Prerequisites

### Required Tools

| Tool | Version | Purpose | Install |
|------|---------|---------|---------|
| Go | 1.22+ | Controller and CLI development | [go.dev](https://go.dev/doc/install) |
| Node.js | 20+ | Console frontend development | [nodejs.org](https://nodejs.org/) |
| Docker | Latest | Container builds, KIND clusters | [docker.com](https://www.docker.com/get-started) |
| kubectl | 1.28+ | Kubernetes interaction | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) |
| kind | Latest | Local test clusters | `go install sigs.k8s.io/kind@latest` |
| helm | 3.x | Chart development | [helm.sh](https://helm.sh/docs/intro/install/) |
| make | Latest | Build automation | Usually pre-installed |

### Optional Tools

| Tool | Purpose |
|------|---------|
| k9s | TUI for Kubernetes |
| stern | Multi-pod log tailing |
| kubebuilder | Controller scaffolding |
| golangci-lint | Go linting |

### Verify Installation

```bash
go version       # go version go1.22.x
node --version   # v20.x.x
docker version   # Client and Server info
kubectl version  # Client Version: v1.28+
kind version     # kind v0.x.x
helm version     # version.BuildInfo{Version:"v3.x.x"...}
```

---

## Repository Setup

### Clone Repositories

```bash
# Create workspace
mkdir -p ~/src/butler && cd ~/src/butler

# Clone core repositories
git clone https://github.com/butlerdotdev/butler-api
git clone https://github.com/butlerdotdev/butler-controller
git clone https://github.com/butlerdotdev/butler-bootstrap
git clone https://github.com/butlerdotdev/butler-cli
git clone https://github.com/butlerdotdev/butler-server
git clone https://github.com/butlerdotdev/butler-console
git clone https://github.com/butlerdotdev/butler-charts
```

### Install Dependencies

#### Go Projects (controller, bootstrap, cli, server)

```bash
cd butler-controller
make deps
# Or: go mod download
```

#### Console Frontend

```bash
cd butler-console
npm install
```

---

## Local Development

### Go Controllers

#### Build

```bash
cd butler-controller
make build
# Binary at bin/manager
```

#### Run Locally

Run against a KIND cluster:

```bash
# Create cluster
kind create cluster --name butler-dev

# Install CRDs
kubectl apply -f ../butler-api/config/crd/bases/

# Run controller
make run
# Or with specific kubeconfig:
# go run ./cmd/main.go --kubeconfig ~/.kube/config
```

#### Code Generation

After modifying `*_types.go` files:

```bash
# Generate DeepCopy methods
make generate

# Update CRD manifests
make manifests
```

### CLI Development

```bash
cd butler-cli

# Build both CLIs
make build

# Run directly
go run ./cmd/butlerctl/main.go cluster list
go run ./cmd/butleradm/main.go version

# Install locally
make install
# Installs to $GOPATH/bin
```

### Console Frontend

```bash
cd butler-console

# Development server (hot reload)
npm run dev
# Access at http://localhost:5173

# Build for production
npm run build

# Preview production build
npm run preview
```

### Server Backend

```bash
cd butler-server

# Build
make build

# Run locally
make run
# Or with environment:
# BUTLER_KUBECONFIG=~/.kube/config go run ./cmd/server/main.go

# Server runs on http://localhost:8080
```

---

## Testing

### Unit Tests

#### Go Projects

```bash
# Run all tests
make test

# With coverage
make test-coverage
go tool cover -html=coverage.out

# Specific package
go test ./internal/controller/... -v
```

#### Frontend

```bash
cd butler-console

# Run tests
npm test

# With coverage
npm run test:coverage
```

### Integration Tests (envtest)

Controllers use envtest for integration testing:

```bash
cd butler-controller

# Install envtest binaries
make envtest

# Run integration tests
make test
```

### E2E Tests

```bash
cd butler-controller

# Create KIND cluster with CRDs
make e2e-setup

# Run E2E tests
make e2e-test

# Cleanup
make e2e-cleanup
```

### Manual Testing

```bash
# Create KIND cluster
kind create cluster --name butler-test

# Install CRDs
kubectl apply -f butler-api/config/crd/bases/

# Install controller (from local build)
make docker-build IMG=butler-controller:dev
kind load docker-image butler-controller:dev --name butler-test
kubectl apply -f config/manager/manager.yaml

# Create test resources
kubectl apply -f config/samples/
```

---

## Debugging

### VS Code Launch Configuration

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Controller",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/cmd/main.go",
      "args": ["--kubeconfig", "${env:HOME}/.kube/config"],
      "env": {
        "ENABLE_WEBHOOKS": "false"
      }
    },
    {
      "name": "Debug CLI",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/cmd/butlerctl/main.go",
      "args": ["cluster", "list"]
    }
  ]
}
```

### Delve (CLI Debugging)

```bash
# Install delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug controller
dlv debug ./cmd/main.go -- --kubeconfig ~/.kube/config

# Debug test
dlv test ./internal/controller/tenantcluster/
```

### Logging

Controllers use zap logging. Increase verbosity:

```bash
# Run with debug logging
go run ./cmd/main.go --zap-log-level=debug

# Or set environment
export LOG_LEVEL=debug
make run
```

### Kubernetes Resources

```bash
# Watch CRD status
kubectl get tenantcluster -A -w

# Describe for events
kubectl describe tenantcluster my-cluster -n default

# Controller logs
kubectl logs -n butler-system deploy/butler-controller -f

# Check events
kubectl get events --sort-by='.lastTimestamp' -A
```

---

## Docker Builds

### Build Image

```bash
# Build with default tag
make docker-build

# Custom tag
make docker-build IMG=ghcr.io/butlerdotdev/butler-controller:dev
```

### Multi-Architecture

```bash
# Build for multiple platforms
make docker-buildx IMG=ghcr.io/butlerdotdev/butler-controller:dev \
  PLATFORMS=linux/amd64,linux/arm64
```

### Load to KIND

```bash
kind load docker-image butler-controller:dev --name butler-test
```

---

## Helm Development

### Lint Charts

```bash
cd butler-charts/charts/butler-controller
helm lint .
```

### Template Rendering

```bash
helm template butler-controller . \
  --namespace butler-system \
  --set image.tag=dev
```

### Local Install

```bash
# From chart directory
helm upgrade --install butler-controller . \
  --namespace butler-system \
  --create-namespace \
  --set image.tag=dev
```

---

## Common Tasks

### Update Dependencies

```bash
# Go modules
go get -u ./...
go mod tidy

# npm
npm update
```

### Add New CRD

1. Edit types in `butler-api/api/v1alpha1/`
2. Run `make generate manifests` in butler-api
3. Run CRD sync workflow
4. Update controller to reconcile new type

### Add New Command

1. Create command file in `internal/cmd/`
2. Register in root command
3. Add tests
4. Update documentation

---

## Getting Help

- **Questions**: Open a GitHub Discussion
- **Bugs**: Open an issue in the relevant repository
- **Chat**: See community channels in [CONTRIBUTING.md](../../CONTRIBUTING.md)
