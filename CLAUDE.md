# talos-operator -- Claude Code Configuration

Kubernetes operator for managing Talos Linux clusters. Built with kubebuilder and controller-runtime.

- **Language**: Go 1.24
- **Framework**: kubebuilder / controller-runtime v0.19.1
- **Module**: `github.com/alperencelik/talos-operator`
- **Container Image**: `alperencelik/talos-operator:latest`

## Architecture

```
cmd/main.go                  -> Operator entrypoint
api/v1alpha1/                -> CRD type definitions
internal/controller/         -> Reconciler implementations
pkg/talos/                   -> Talos client, config bundling, machine operations
pkg/storage/                 -> S3 backup storage
pkg/utils/                   -> Shared utilities
config/crd/bases/            -> Generated CRD manifests
config/rbac/                 -> RBAC manifests
deploy/talos-operator/       -> Helm chart
```

## Custom Resources (v1alpha1)

- **TalosCluster** -- Represents a Talos Linux cluster
- **TalosControlPlane** -- Manages control plane nodes (metal + cloud modes)
- **TalosWorker** -- Manages worker nodes
- **TalosMachine** -- Individual machine configuration and lifecycle
- **TalosEtcdBackup** -- Etcd backup schedule and S3 storage

## Build & Development Commands

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

## Testing

- **Framework**: Ginkgo v2 + Gomega
- **Unit tests**: `make test` (uses envtest for K8s API)
- **E2E tests**: `make test-e2e` (requires Kind cluster)
- **Test files**: `*_test.go` alongside source in `internal/controller/`

## Code Generation

After modifying CRD types in `api/v1alpha1/`:
1. `make generate` -- regenerate `zz_generated.deepcopy.go`
2. `make manifests` -- regenerate CRD YAML in `config/crd/bases/`
3. `make chart` -- sync CRDs to Helm chart

## Dependencies

Key external dependencies:
- `siderolabs/talos` -- Talos Linux API and machinery
- `siderolabs/go-kubernetes` -- Kubernetes helpers for Talos
- `aws/aws-sdk-go-v2` -- S3 storage for etcd backups
- `controller-runtime` -- Kubernetes controller framework
- `onsi/ginkgo` + `onsi/gomega` -- Test framework

## Talos API Patterns

The operator interacts with Talos nodes via the Talos gRPC API (`pkg/talos/`):

### Client Operations (`pkg/talos/client.go`)
- `NewClient()` -- Create Talos API client for a node endpoint
- `ApplyConfig()` -- Apply machine configuration to a node
- `ApplyMetaKey()` -- Set META key values (e.g., 0x0a for network config)
- `Bootstrap()` -- Bootstrap etcd on the first control plane node
- `GetMemberStatus()` -- Check etcd membership and health
- `Upgrade()` -- Trigger Talos/K8s version upgrade

### Config Bundling (`pkg/talos/bundle.go`)
- `GenerateBundle()` -- Create Talos secrets bundle for a cluster
- Patch templates: `InstallDiskPatch`, `InstallImagePatch`, `StaticNetworkPatch`, `VIPExtension`
- `VolumeConfigTemplate` -- VolumeConfig for data disk mounting

### Machine Config Generation (`pkg/talos/metakey_tpl.go`)
- META key 0x0a network template for pre-install networking
- Renders minimal static IP config for maintenance mode reachability

## Cluster Bootstrap Lifecycle

```
1. TalosControlPlane reconciled
   -> Create TalosMachine CRs for each CP node (from metalSpec.machines)
   -> Assign IPs via networkAllocation (if specified)

2. First TalosMachine reconciled (CP node 0)
   -> Apply META key (pre-install network)
   -> Generate + apply machine config (control plane)
   -> Bootstrap etcd

3. Additional CP TalosMachines reconciled
   -> Apply META key + machine config
   -> Join existing etcd cluster

4. TalosWorker reconciled (after CP is healthy)
   -> Create TalosMachine CRs for workers
   -> Apply META key + machine config (worker role)
   -> Nodes join the cluster

5. Ongoing reconciliation
   -> Monitor machine health, handle upgrades
   -> Etcd backups via TalosEtcdBackup
```

## Network Configuration

### Two-Layer Static IP Approach

**Layer 1 -- META key** (pre-install): Minimal network config applied before machine config. Sets a reachable IP in maintenance mode.

**Layer 2 -- Machine config patch** (post-install): Full production network config using `deviceSelector.hardwareAddr` (MAC) or `deviceSelector.busPath` (PCI). Persists across reboots.

### Key Types
- `NetworkSpec` -- Static IP, CIDR, gateway, MAC, nameservers, VIP, multi-interface support
- `InterfaceSpec` -- Per-interface config with MAC/busPath selectors
- `NetworkAllocation` -- Deterministic IP + VMID assignment from TalosControlPlane

---

# Principles

## Compile First (CRITICAL)

Code MUST compile before any other quality check. Compiler errors are not warnings.

```
COMPILE -> LINT -> TEST -> COMMIT
```

- Every code edit triggers `go build ./...`
- ALL compiler errors must be fixed before proceeding
- Common violations: unused imports, undefined variables, type mismatches, missing returns

## Strict Linting (CRITICAL)

Lint violations are treated as compilation errors. Zero tolerance.

- **Linter**: `golangci-lint` with `enable-all: true`
- **Static analysis**: `staticcheck` all checks enabled
- **Error handling**: `errcheck` with `check-type-assertions: true`
- **Security**: `gosec` with high severity
- Suppressions require documented justification: `//nolint:errcheck // reason: ...`

## No Attribution (CRITICAL)

No output may contain strings that identify code as AI-produced. No "generated by", "AI-generated", "co-authored-by" tags, or similar markers in code, comments, commits, or docs.

## Engineering Mindset

- **SOLID**: Single responsibility, open/closed, Liskov, interface segregation, dependency inversion
- **DRY**: Abstract common functionality, eliminate duplication
- **KISS**: Prefer simplicity over complexity
- **YAGNI**: Implement current requirements only, avoid speculation

---

# Rules

## Workflow
- **Pattern**: Understand -> Plan -> Execute -> Validate
- **Parallel Operations**: Batch independent tool calls, sequential only for dependencies
- **Validation Gates**: Run lint/typecheck before marking tasks complete

## Implementation Completeness
- No partial features: if you start, you MUST complete to working state
- No TODO comments for core functionality
- No mock objects, placeholders, or stub implementations
- Every function must work as specified

## Scope Discipline
- Build ONLY what is asked -- no adding features beyond explicit requirements
- MVP first, iterate based on feedback
- Single responsibility per component

## Git Workflow
- Always check `git status` and `git branch` first
- Feature branches for ALL work, never commit to main/master
- Incremental commits with meaningful messages
- `git diff` before staging

## Failure Investigation
- Root cause analysis: always investigate WHY failures occur
- Never disable, comment out, or skip tests
- Debug systematically: Understand -> Diagnose -> Fix -> Verify

## Code Organization
- Follow existing project naming conventions
- `zz_generated.deepcopy.go` is auto-generated -- never modify
- CRD changes require the full code generation pipeline
- Follow existing controller patterns in `internal/controller/`

---

# Quality Pipeline

```
Edit/Write operation
  -> Compile: go build ./...              (hard-block)
  -> Lint: golangci-lint run              (hard-block)
  -> Vet: go vet ./...                    (validation)
  -> Test: go test ./... -race -count=1   (validation)
  -> All pass -> Code accepted
```

## Go-Specific Quality Gates

| Check | Command | Blocks? |
|---|---|---|
| Compilation | `go build ./...` | Yes -- nothing runs if broken |
| Linting | `golangci-lint run` | Yes -- lint violations are errors |
| Vet | `go vet ./...` | Yes -- catches suspicious constructs |
| Race detection | `go test -race ./...` | Yes -- data races are bugs |
| Code generation | `make generate && make manifests` | Required after CRD changes |

---

# Development Patterns

## Controller-Runtime Reconciliation

```go
func (r *TalosMachineReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the TalosMachine CR
    // 2. Check if deleted -> handle finalizer
    // 3. Determine machine role (CP or worker) from refs
    // 4. Apply META key for pre-install networking
    // 5. Generate and apply machine config with patches
    // 6. Bootstrap etcd (first CP only)
    // 7. Update status subresource
    // 8. Return ctrl.Result{} or requeue
}
```

## Machine Config Patching (`metalConfigPatches`)

Patches are appended as multi-doc YAML to the base machine config:
1. Install disk patch
2. Install image patch
3. Wipe disk patch
4. Air-gap / registries patch
5. Static network patch (from NetworkSpec)
6. VolumeConfig patches (from DataVolumes)

## Status Subresource Updates

Always update status separately from spec changes:
```go
if err := r.Status().Update(ctx, tm); err != nil {
    return ctrl.Result{}, err
}
```

## Error Handling

- Return `ctrl.Result{RequeueAfter: duration}` for transient errors (node not ready, etcd not healthy)
- Return `ctrl.Result{}, err` for unexpected errors (auto-requeue with backoff)
- Return `ctrl.Result{}, nil` when reconciliation is complete

---

# Available Skills

| Skill | Purpose |
|---|---|
| `compile-guard` | Compilation verification before downstream checks |
| `polyglot-lint-enforcement` | Multi-language lint enforcement |
| `feature-dev` | Guided feature development with codebase understanding |
| `openspec-workflow` | Feature implementation via structured spec |
| `systematic-debugging` | Bug investigation with root cause analysis |
| `test-driven-development` | TDD workflow for features and bug fixes |
| `code-review` | Code review for pull requests |
| `commit` | Create well-formed git commits |
| `commit-push-pr` | Commit, push, and open PR |
| `brainstorming` | Interactive requirements discovery |

## Available Agents

| Agent | Domain |
|---|---|
| `golang-expert` | Go development expertise |
| `go-leak-detector` | Goroutine leak analysis |
| `lint-enforcer` | Project-wide lint audits |
| `backend-architect` | Backend system design |
| `security-engineer` | Security vulnerability analysis |
| `quality-engineer` | Testing strategies |
| `system-architect` | System architecture design |
| `performance-engineer` | Performance optimization |
| `refactoring-expert` | Code quality improvement |
| `devops-architect` | Infrastructure and deployment |

## MCP Servers

| Server | Use For |
|---|---|
| Context7 | Go docs, controller-runtime docs, Talos API docs |
| Serena | Symbol navigation, semantic code understanding |
| Sequential | Complex debugging, architectural analysis |

---

# Flags Reference

| Flag | Purpose |
|---|---|
| `--think` | Standard structured analysis (~4K tokens) |
| `--think-hard` | Deep analysis (~10K tokens) |
| `--ultrathink` | Maximum depth analysis (~32K tokens) |
| `--brainstorm` | Collaborative discovery for vague requests |
| `--delegate` | Enable sub-agent parallel processing |
| `--validate` | Pre-execution risk assessment |
