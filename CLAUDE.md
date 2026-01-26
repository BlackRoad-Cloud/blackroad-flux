# CLAUDE.md - AI Assistant Guide for Flux v2

This document provides essential context for AI assistants working with the Flux v2 codebase.

## Project Overview

**Flux v2** is a GitOps tool for keeping Kubernetes clusters in sync with configuration sources (Git repositories, OCI artifacts) and automating continuous delivery. It's a CNCF graduated project built on Kubernetes' API extension system.

- **Module**: `github.com/fluxcd/flux2/v2`
- **Go Version**: 1.25.0
- **License**: BlackRoad OS Proprietary (see LICENSE)

## Architecture

Flux v2 consists of a CLI (`flux`) that orchestrates multiple Kubernetes controllers:

| Controller | Purpose |
|------------|---------|
| source-controller | Manages Git, OCI, Helm, and Bucket sources |
| kustomize-controller | Applies Kustomize overlays to clusters |
| helm-controller | Manages Helm releases |
| notification-controller | Handles events and webhooks |
| image-reflector-controller | Scans container registries |
| image-automation-controller | Updates image tags in Git |

## Directory Structure

```
cmd/flux/           # CLI implementation (Cobra commands)
pkg/                # Reusable business logic
  bootstrap/        # Cluster bootstrapping
  manifestgen/      # Manifest generation
  status/           # Resource health assessment
  uninstall/        # Flux removal
  log/              # Logger interface
  printers/         # Output formatters
internal/           # Internal packages
  build/            # Kustomize building, dry-run
  utils/            # Common utilities
  flags/            # CLI flag definitions
  tree/             # Resource tree visualization
manifests/          # Kubernetes manifests for controllers
tests/              # Integration and e2e tests
.github/workflows/  # CI/CD pipelines
```

## Development Setup

### Prerequisites

- Go >= 1.25
- kubectl >= 1.30
- kustomize >= 5.0
- kind (for e2e tests)

### Build Commands

```bash
# Build flux binary
make build

# Build development version (includes git info)
make build-dev

# Install to $GOPATH/bin
make install

# Format code
make fmt

# Run vet
make vet

# Tidy dependencies
make tidy
```

### Testing

```bash
# Install envtest binaries (required for unit tests)
make install-envtest

# Run unit tests
make test

# Run specific package tests
make test TEST_PKG_PATH="./cmd/flux"

# Setup KinD cluster for e2e
make setup-kind

# Run e2e tests
make e2e

# Run e2e with specific package
make e2e E2E_TEST_PKG_PATH="./cmd/flux"

# Update golden files (when CLI output changes)
make test TEST_PKG_PATH="./cmd/flux" TEST_ARGS="-update"
make e2e E2E_TEST_PKG_PATH="./cmd/flux" TEST_ARGS="-update"

# Cleanup KinD cluster
make cleanup-kind

# Full e2e cycle
make test-with-kind
```

## Code Patterns

### CLI Command Structure

Each command follows this pattern in `cmd/flux/`:

```go
var commandCmd = &cobra.Command{
    Use:     "command [flags]",
    Short:   "Short description",
    Long:    "Long description with examples",
    Example: "flux command --flag value",
    RunE:    commandCmdRun,
}

type commandFlags struct {
    flagName string
}

var commandArgs commandFlags

func init() {
    commandCmd.Flags().StringVar(&commandArgs.flagName, "flag", "", "description")
    parentCmd.AddCommand(commandCmd)
}

func commandCmdRun(cmd *cobra.Command, args []string) error {
    // Implementation
}
```

### Interface Adapters

The codebase uses adapter patterns for Kubernetes resources:

```go
// Interfaces in cmd/flux/main.go
type adapter interface { ... }
type named interface { ... }
type copyable interface { ... }
type upsertable interface { ... }
```

### Error Handling

Custom error types with status codes:

```go
type RequestError struct {
    StatusCode int
    Err        error
}
```

### Golden File Testing

E2E tests use golden files for output comparison:
- Golden files: `cmd/flux/testdata/*/create_*.golden`
- Update with `-update` flag when CLI output changes intentionally

## Key Files

| File | Purpose |
|------|---------|
| `cmd/flux/main.go` | Entry point, version constant, root command |
| `cmd/flux/root.go` | Root command, global flags |
| `pkg/bootstrap/bootstrap.go` | Core bootstrapping logic |
| `internal/build/build.go` | Kustomize build orchestration |
| `Makefile` | Build and test targets |
| `.goreleaser.yml` | Multi-platform release config |

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `FLUX_SYSTEM_NAMESPACE` | Default namespace (flux-system) |
| `TEST_KUBECONFIG` | Kubeconfig for e2e tests |
| `KUBEBUILDER_ASSETS` | Path to envtest binaries |
| `ENVTEST_KUBERNETES_VERSION` | K8s version for envtest |

## Common Tasks

### Adding a New Command

1. Create `cmd/flux/command_name.go`
2. Follow the command structure pattern above
3. Register with parent command in `init()`
4. Add tests in `cmd/flux/command_name_test.go`
5. Add golden files if testing CLI output

### Updating Controller Dependencies

When controller APIs change:
1. Update versions in `go.mod`
2. Run `make tidy`
3. Update manifests if CRDs changed
4. Run `make test` to verify

### Adding New Flags

1. Add flag to appropriate `*Flags` struct
2. Register in `init()` with appropriate type method
3. Use in command's `RunE` function

## CI/CD Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `e2e.yaml` | Push/PR to main | Unit tests, e2e tests, CLI verification |
| `release.yaml` | Git tags `v*` | Multi-platform builds, signing, release |
| `update.yaml` | Scheduled | Update controller dependencies |
| `conformance.yaml` | PR | Kubernetes conformance tests |
| `scan.yaml` | Push/PR | Security scanning (FOSSA, Snyk, CodeQL) |

## Commit Guidelines

- Limit subject to 50 characters
- Write as continuation of "If applied, this commit will..."
- Explain what and why in body (wrap at 72 chars)
- Sign commits with `-s` flag (DCO required)

## Important Notes for AI Assistants

1. **Always run tests** after making changes: `make test`
2. **Check golden files** when modifying CLI output
3. **Keep cross-platform compatibility** - code must build on Linux and Darwin
4. **Follow Go conventions**: gofmt, govet, code review comments
5. **Resource names** must be RFC 1123 compliant (DNS labels)
6. **No external state** - all state is in Kubernetes resources
7. **Context-aware operations** - respect namespace and kubeconfig flags

## Quick Reference

```bash
# Verify build
make build && ./bin/flux version

# Full validation before PR
make tidy && make test && make e2e

# Check specific component
./bin/flux check --pre

# Debug failing tests
kubectl -n flux-system get all
kubectl -n flux-system describe pods
kubectl -n flux-system logs deploy/source-controller
```
