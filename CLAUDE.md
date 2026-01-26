# CLAUDE.md - AI Assistant Guide for Flux v2

This document provides comprehensive guidance for AI assistants working with the Flux v2 codebase.

## Project Overview

**Flux v2** is a Kubernetes-native GitOps continuous delivery tool and CNCF graduated project. It keeps Kubernetes clusters in sync with configuration sources (Git repositories, OCI artifacts, Helm charts) and automates updates when new code is deployed.

- **Language:** Go 1.25
- **Module:** `github.com/fluxcd/flux2/v2`
- **CLI Binary:** `flux`
- **Architecture:** Kubernetes controller pattern with CRDs

## Directory Structure

```
blackroad-flux/
├── cmd/flux/              # CLI command implementations (~230 Go files)
│   ├── main.go            # Entry point, version info
│   ├── create_*.go        # Resource creation commands
│   ├── delete_*.go        # Resource deletion commands
│   ├── get_*.go           # Resource listing commands
│   ├── reconcile_*.go     # Reconciliation triggers
│   └── bootstrap_*.go     # Cluster bootstrap commands
├── pkg/                   # Public packages (importable by other projects)
│   ├── bootstrap/         # Bootstrap orchestration logic
│   ├── manifestgen/       # Manifest generation (install, kustomization, secrets)
│   ├── status/            # Health status checking and polling
│   ├── printers/          # Output formatters (YAML, table, diff)
│   ├── log/               # Logging interfaces
│   └── uninstall/         # Uninstall logic
├── internal/              # Internal packages (not importable externally)
│   ├── build/             # Kustomization building and dry-run
│   ├── flags/             # CLI flag parsing and validation types
│   ├── tree/              # Resource tree rendering
│   └── utils/             # Utilities (kubectl, YAML handling)
├── manifests/             # Kubernetes manifests for Flux deployment
│   ├── bases/             # Base controller manifests
│   ├── crds/              # Custom Resource Definitions
│   ├── install/           # Installation kustomization
│   ├── rbac/              # RBAC policies
│   ├── policies/          # Network policies
│   └── scripts/           # Manifest bundling scripts
├── tests/                 # Integration tests
│   └── integration/       # E2E test suites with terraform
├── .github/               # GitHub Actions CI/CD workflows
│   ├── workflows/         # CI/CD workflow definitions
│   └── kind/              # KinD cluster configuration
├── rfcs/                  # Request for Comments documentation
└── docs/                  # Additional documentation
```

## Key Packages

### cmd/flux/ - CLI Commands

The CLI uses Cobra framework. Commands follow a hierarchical pattern:

```
flux <root-command> <sub-command> [arguments] [flags]
```

**Major command groups:**
- `bootstrap` - Deploy Flux on cluster (github, gitlab, gitea, bitbucket-server)
- `create` - Create/update Flux resources
- `delete` - Remove resources
- `get` - List resources with status
- `reconcile` - Trigger resource reconciliation
- `install/uninstall` - Deploy/remove Flux manifests
- `export` - Output resources as YAML
- `suspend/resume` - Control resource sync
- `build/diff` - Preview changes before applying
- `logs/trace/events` - Debugging and monitoring

### pkg/bootstrap/

Orchestrates Flux installation on Kubernetes clusters:
- `Reconciler` interface for component reconciliation
- Provider implementations (GitHub, GitLab, Gitea, etc.)
- Secret and sync configuration management

### pkg/manifestgen/

Generates Kubernetes manifests:
- `install/` - Installation manifests (kustomize-based)
- `kustomization/` - Kustomization CRD generation
- `sourcesecret/` - Git credential secrets
- `sync/` - GitOps sync manifests

### pkg/status/

Health monitoring:
- `StatusChecker` - Polls resource conditions
- Uses `cli-utils` for status aggregation
- Monitors Ready conditions across resources

### internal/build/

Kustomization building:
- Dry-run execution
- Diff computation between manifests

### internal/flags/

Custom flag types with validation:
- `LogLevel`, `PublicKeyAlgorithm`, `RsaKeyBits`
- `SafeRelativePath`, `HelmChartSource`
- Git/OCI/Bucket source providers

## Build System

### Essential Make Targets

```bash
# Development
make build          # Build flux binary with version
make build-dev      # Build with dev version (branch-sha-timestamp)
make install        # Install flux CLI locally
make tidy           # Tidy go.mod/go.sum

# Testing
make test           # Unit tests with envtest + fmt + vet
make e2e            # End-to-end tests (requires KinD cluster)
make test-with-kind # Full integration: setup + e2e + cleanup

# Test Infrastructure
make setup-kind     # Create KinD cluster with Calico
make cleanup-kind   # Destroy KinD cluster
make install-envtest # Setup controller-runtime envtest
```

### Build Configuration

```bash
# Version from main.go
VERSION=$(grep 'VERSION' cmd/flux/main.go | awk '{ print $4 }' | head -n 1 | tr -d '"')

# Dev version format
DEV_VERSION=0.0.0-$(branch)-$(sha)-$(timestamp)

# Build command
CGO_ENABLED=0 go build -ldflags="-s -w -X main.VERSION=$(VERSION)" -o ./bin/flux ./cmd/flux
```

### Manifest Bundling

Manifests are embedded into the binary via `manifests/scripts/bundle.sh`. The marker file `cmd/flux/.manifests.done` tracks build dependencies.

## Testing

### Prerequisites

- Go >= 1.25
- kubectl >= 1.30
- kustomize >= 5.0
- KinD (for E2E tests)

### Unit Tests

```bash
# Run all unit tests
make test

# Run specific package tests
make test TEST_PKG_PATH="./cmd/flux"

# Update golden files (for CLI output tests)
make test TEST_PKG_PATH="./cmd/flux" TEST_ARGS="-update"
```

Tests require `KUBEBUILDER_ASSETS` environment variable pointing to envtest binaries.

### E2E Tests

```bash
# Setup test cluster
make setup-kind

# Run E2E tests
make e2e

# Run specific E2E package
make e2e E2E_TEST_PKG_PATH="./cmd/flux" TEST_ARGS="-update"

# Cleanup
make cleanup-kind
```

### Test File Conventions

- Unit tests: `*_test.go` adjacent to source files
- Build tags: `//go:build unit` for unit tests, `//go:build e2e` for E2E
- Golden files in `testdata/` directories
- Table-driven tests preferred

## Code Conventions

### File Organization

- One command per file: `cmd/flux/create_source_git.go`
- Initialization in `init()` function: register command, setup flags
- Use `RunE` for commands that can fail

### Naming Conventions

```go
// Commands: lowercase with underscores in filenames
create_source_git.go

// Flags: lowercase with hyphens
--api-server, --kubeconfig

// Exported functions: PascalCase
func GenerateManifests() error

// Interfaces: descriptive, often ending in -er
type Printer interface { ... }
type Reconciler interface { ... }
```

### Import Organization

```go
import (
    // Standard library
    "context"
    "fmt"

    // Third-party
    "github.com/spf13/cobra"

    // Kubernetes
    "k8s.io/client-go/..."

    // Flux packages
    "github.com/fluxcd/flux2/v2/pkg/..."
)
```

### Error Handling

```go
// Wrap errors with context
if err != nil {
    return fmt.Errorf("failed to create resource: %w", err)
}

// Validate before operations
if name == "" {
    return fmt.Errorf("name is required")
}
```

### Logging

```go
// Use the custom logger from rootCmd
logger.Successf("resource created")
logger.Failuref("operation failed: %s", err)
logger.Warningf("deprecated flag used")
```

## Common Flag Patterns

```bash
-n, --namespace string      # Kubernetes namespace (default: flux-system)
--kubeconfig string         # Path to kubeconfig file
-v, --verbose               # Print generated manifests
--export                    # Output YAML instead of applying
--timeout duration          # Operation timeout (default: 5m)
--context string            # Kubernetes context to use
```

## External Dependencies

### Flux Controller APIs

```go
github.com/fluxcd/source-controller/api      # GitRepository, HelmRepository, etc.
github.com/fluxcd/kustomize-controller/api   # Kustomization
github.com/fluxcd/helm-controller/api        # HelmRelease
github.com/fluxcd/notification-controller/api # Provider, Alert, Receiver
github.com/fluxcd/image-*-controller/api     # Image automation
```

### Key Libraries

```go
github.com/spf13/cobra          # CLI framework
github.com/fluxcd/pkg/...       # Shared Flux utilities
sigs.k8s.io/controller-runtime  # Kubernetes controller patterns
github.com/homeport/dyff        # YAML diff visualization
```

## Common Development Tasks

### Adding a New Command

1. Create file `cmd/flux/<action>_<resource>.go`
2. Define command struct and flags
3. Register in `init()` with parent command
4. Implement `RunE` function
5. Add corresponding test file

### Adding a New Flag Type

1. Create type in `internal/flags/`
2. Implement `pflag.Value` interface (`String()`, `Set()`, `Type()`)
3. Add validation in `Set()` method

### Modifying Manifests

1. Update YAML in `manifests/` directory
2. Run `make build` to re-bundle manifests
3. Test with `make e2e`

### Updating Controller APIs

1. Update version in `go.mod`
2. Run `make tidy`
3. Update any affected command implementations
4. Run tests

## Commit Message Format

```
Limit subject to 50 characters

If applied, this commit will <subject>

Explain what and why in the body (wrap at 72 chars)
```

**Examples:**
- `Add support for OCI repository authentication`
- `Fix timeout handling in reconcile command`
- `Update source-controller API to v1.7.4`

## DCO Sign-off

All commits must be signed off:

```bash
git commit -s -m "Your commit message"
# Produces: Signed-off-by: Name <email>
```

## CI/CD Workflows

Key GitHub Actions workflows in `.github/workflows/`:

- `e2e.yaml` - Unit tests and E2E on KinD
- `e2e-azure.yaml`, `e2e-gcp.yaml` - Cloud provider tests
- `release.yaml` - Build and publish releases
- `conformance.yaml` - Kubernetes conformance tests
- `scan.yaml` - Security scanning

## Quick Reference

```bash
# Build and test locally
make build-dev && ./bin/flux version
make test

# Check prerequisites
./bin/flux check --pre

# Common flux commands
flux bootstrap github --owner=<org> --repository=<repo>
flux get sources git -A
flux reconcile kustomization <name> --with-source
flux logs --follow
```

## Troubleshooting

### Test Failures

- Ensure `KUBEBUILDER_ASSETS` is set: `make install-envtest`
- For E2E: ensure KinD cluster is running: `make setup-kind`

### Build Issues

- Run `make tidy` to fix module issues
- Ensure Go 1.25+ is installed
- Check manifest bundling: `rm cmd/flux/.manifests.done && make build`

### Import Errors

- Controller APIs are in separate repos; check versions in `go.mod`
- Use `go mod tidy -compat=1.25` after dependency changes
