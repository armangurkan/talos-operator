# ═══════════════════════════════════════════════════
# talos-operator — Claude Code Configuration
# ═══════════════════════════════════════════════════

# Project Overview

Kubernetes operator for managing Talos Linux clusters. Built with kubebuilder and controller-runtime.

- **Language**: Go 1.24
- **Framework**: kubebuilder / controller-runtime v0.19.1
- **Module**: `github.com/alperencelik/talos-operator`
- **Container Image**: `alperencelik/talos-operator:latest`

# Architecture

```
cmd/main.go                  → Operator entrypoint
api/v1alpha1/                → CRD type definitions (TalosCluster, TalosControlPlane, TalosWorker, TalosMachine, TalosEtcdBackup)
internal/controller/         → Reconciler implementations
pkg/talos/                   → Talos client, config bundling, machine operations
pkg/storage/                 → S3 backup storage
pkg/utils/                   → Shared utilities
config/crd/bases/            → Generated CRD manifests
config/rbac/                 → RBAC manifests
deploy/talos-operator/       → Helm chart
```

# Custom Resources (v1alpha1)

- **TalosCluster** — Represents a Talos Linux cluster
- **TalosControlPlane** — Manages control plane nodes
- **TalosWorker** — Manages worker nodes
- **TalosMachine** — Individual machine configuration
- **TalosEtcdBackup** — Etcd backup schedule and storage

# Build & Development Commands

```bash
make build          # Build manager binary (includes manifests, generate, fmt, vet)
make test           # Run unit tests with envtest (K8s 1.31.0)
make test-e2e       # Run e2e tests (requires Kind cluster)
make lint           # Run golangci-lint
make lint-fix       # Run golangci-lint with --fix
make manifests      # Regenerate CRDs, RBAC, webhooks
make generate       # Regenerate DeepCopy methods
make chart          # Copy/clean CRDs for Helm chart
make docker-build   # Build container image
make install        # Install CRDs into cluster
make deploy         # Deploy operator to cluster
```

# Testing

- **Framework**: Ginkgo v2 + Gomega
- **Unit tests**: `make test` (uses envtest for K8s API)
- **E2E tests**: `make test-e2e` (requires Kind cluster)
- **Test files**: `*_test.go` alongside source in `internal/controller/`

# Code Generation

After modifying CRD types in `api/v1alpha1/`:
1. `make generate` — regenerate `zz_generated.deepcopy.go`
2. `make manifests` — regenerate CRD YAML in `config/crd/bases/`
3. `make chart` — sync CRDs to Helm chart

# Principles

- CompileFirst: Always ensure `go build ./...` passes before committing
- StrictLinting: All code must pass `make lint` (golangci-lint)
- Never modify `zz_generated.deepcopy.go` — it is auto-generated
- Follow existing controller patterns in `internal/controller/`
- CRD changes require running the full code generation pipeline

# Dependencies

Key external dependencies:
- `siderolabs/talos` — Talos Linux API and machinery
- `siderolabs/go-kubernetes` — Kubernetes helpers for Talos
- `aws/aws-sdk-go-v2` — S3 storage for etcd backups
- `controller-runtime` — Kubernetes controller framework
- `onsi/ginkgo` + `onsi/gomega` — Test framework
