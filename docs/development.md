# Development Guide

## Prerequisites

### Required Software

| Software | Version | Purpose |
|----------|---------|---------|
| Go | 1.21+ | Primary language |
| Make | 4.0+ | Build automation |
| Docker | 24.0+ | Container builds |
| Git | 2.40+ | Version control |
| protoc | 3.21+ | Protocol buffers |
| Node.js | 18+ | Web UI development |

### Optional Tools

| Tool | Purpose |
|------|---------|
| golangci-lint | Code linting |
| mockgen | Mock generation |
| ko | Container image building |
| goreleaser | Release automation |
| act | Local GitHub Actions |

## Getting Started

### Clone Repository

```bash
git clone https://github.com/go-proxmox/gpve.git
cd gpve
````

### Install Dependencies

```bash
# Go dependencies
go mod download

# Development tools
make tools

# Web UI dependencies
cd web && npm install && cd ..
```

### Build

```bash
# Build all binaries
make build

# Build specific binary
make build-server
make build-agent
make build-cli

# Build with version info
make build VERSION=1.0.0 GIT_COMMIT=$(git rev-parse HEAD)
```

### Run Tests

```bash
# All tests
make test

# With coverage
make test-coverage

# Integration tests (requires QEMU)
make test-integration

# Specific package
go test -v ./internal/qemu/...
```

---

## Project Structure

```
gpve/
├── cmd/                        # Application entrypoints
│   ├── gpve-server/           # API server
│   ├── gpve-agent/            # Node agent
│   └── gpvectl/               # CLI tool
│
├── api/                        # API definitions
│   ├── openapi/               # OpenAPI specs
│   └── proto/                 # Protocol buffer definitions
│
├── internal/                   # Private packages
│   ├── api/                   # API handlers
│   │   ├── handlers/          # HTTP handlers
│   │   ├── middleware/        # HTTP middleware
│   │   └── grpc/              # gRPC services
│   │
│   ├── auth/                  # Authentication & authorization
│   │   ├── providers/         # Auth providers (PAM, LDAP, OIDC)
│   │   ├── rbac/              # Role-based access control
│   │   └── token/             # Token management
│   │
│   ├── cluster/               # Cluster management
│   │   ├── consensus/         # Raft/etcd integration
│   │   ├── membership/        # Node membership
│   │   └── scheduler/         # Resource scheduler
│   │
│   ├── compute/               # Compute abstractions
│   │   ├── guest/             # Guest interface
│   │   ├── vm/                # VM implementation
│   │   └── vcluster/          # vCluster implementation
│   │
│   ├── qemu/                  # QEMU integration
│   │   ├── config/            # VM configuration
│   │   ├── qmp/               # QMP client
│   │   ├── monitor/           # VM monitoring
│   │   └── migration/         # Live migration
│   │
│   ├── storage/               # Storage management
│   │   ├── driver/            # Storage drivers
│   │   ├── volume/            # Volume management
│   │   └── backup/            # Backup management
│   │
│   ├── network/               # Network management
│   │   ├── bridge/            # Bridge management
│   │   ├── firewall/          # Firewall (nftables)
│   │   └── sdn/               # Software-defined networking
│   │
│   └── pkg/                   # Shared internal packages
│       ├── config/            # Configuration
│       ├── logger/            # Logging
│       └── utils/             # Utilities
│
├── pkg/                        # Public packages
│   ├── client/                # Go client library
│   └── types/                 # Shared types
│
├── web/                        # Web UI (Vue.js)
│   ├── src/
│   │   ├── components/
│   │   ├── views/
│   │   ├── stores/
│   │   └── api/
│   └── package.json
│
├── deploy/                     # Deployment configurations
│   ├── docker/                # Docker files
│   ├── systemd/               # Systemd units
│   └── kubernetes/            # K8s manifests
│
├── docs/                       # Documentation
├── scripts/                    # Build & utility scripts
├── test/                       # Test fixtures & e2e tests
│   ├── e2e/
│   ├── fixtures/
│   └── mocks/
│
├── Makefile
├── go.mod
├── go.sum
└── README.md
```

---

## Coding Standards

### Go Style Guide

We follow the standard Go style guidelines with some additions:

```go
// Package documentation
// Package qemu provides QEMU/KVM virtual machine management.
package qemu

import (
    // Standard library imports first
    "context"
    "fmt"
    "time"
    
    // Third-party imports second
    "github.com/sirupsen/logrus"
    
    // Project imports last
    "github.com/go-proxmox/gpve/internal/pkg/config"
)

// Constants grouped by purpose
const (
    // DefaultMemory is the default VM memory in MB
    DefaultMemory = 512
    
    // MaxCPUs is the maximum number of vCPUs
    MaxCPUs = 512
)

// VMConfig represents virtual machine configuration.
// 
// Fields are organized by category: identity, compute, storage, network.
type VMConfig struct {
    // Identity
    VMID uint64 `json:"vmid"`
    Name string `json:"name"`
    
    // Compute
    Cores   int    `json:"cores"`
    Sockets int    `json:"sockets"`
    Memory  uint64 `json:"memory"`
    
    // Storage
    Disks []DiskConfig `json:"disks,omitempty"`
    
    // Network
    Networks []NetworkConfig `json:"networks,omitempty"`
}

// Validate checks if the configuration is valid.
//
// It returns an error if any field has an invalid value.
func (c *VMConfig) Validate() error {
    if c.VMID < 100 {
        return fmt.Errorf("vmid must be >= 100, got %d", c.VMID)
    }
    if c.Cores < 1 || c.Cores > MaxCPUs {
        return fmt.Errorf("cores must be between 1 and %d", MaxCPUs)
    }
    return nil
}

// Manager handles VM lifecycle operations.
type Manager struct {
    config    *config.Config
    qmp       *QMPClient
    logger    *logrus.Entry
    
    mu sync.RWMutex // Protects running
    running map[uint64]*RunningVM
}

// NewManager creates a new VM manager.
func NewManager(cfg *config.Config, logger *logrus.Logger) *Manager {
    return &Manager{
        config:  cfg,
        logger:  logger.WithField("component", "vm-manager"),
        running: make(map[uint64]*RunningVM),
    }
}

// Start starts a virtual machine.
//
// It connects to QEMU via QMP and sends the cont command.
// The context can be used to cancel the operation.
func (m *Manager) Start(ctx context.Context, vmid uint64) error {
    m.logger.WithField("vmid", vmid).Info("Starting VM")
    
    // Implementation...
    
    return nil
}
```

### Error Handling

```go
// Define package-level errors
var (
    ErrVMNotFound      = errors.New("vm not found")
    ErrVMAlreadyExists = errors.New("vm already exists")
    ErrVMLocked        = errors.New("vm is locked")
)

// Use error wrapping for context
func (m *Manager) Get(ctx context.Context, vmid uint64) (*VM, error) {
    vm, err := m.store.Get(ctx, vmid)
    if err != nil {
        if errors.Is(err, storage.ErrNotFound) {
            return nil, fmt.Errorf("vmid %d: %w", vmid, ErrVMNotFound)
        }
        return nil, fmt.Errorf("failed to get vm %d: %w", vmid, err)
    }
    return vm, nil
}

// Check errors with errors.Is or errors.As
func handleError(err error) {
    if errors.Is(err, ErrVMNotFound) {
        // Handle not found
    }
    
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        // Handle validation error
    }
}
```

### Context Usage

```go
// Always accept context as first parameter
func (m *Manager) Create(ctx context.Context, cfg *VMConfig) (*VM, error) {
    // Check for cancellation
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }
    
    // Use context for timeouts
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    // Pass context to downstream calls
    if err := m.storage.Allocate(ctx, cfg.Disks); err != nil {
        return nil, err
    }
    
    return vm, nil
}
```

### Logging

```go
// Use structured logging
logger := logrus.WithFields(logrus.Fields{
    "component": "qemu",
    "vmid":      vmid,
})

// Log levels:
// - Debug: Detailed debugging information
// - Info: General operational information
// - Warn: Something unexpected but not an error
// - Error: Error that should be addressed

logger.Debug("Connecting to QMP socket")
logger.Info("VM started successfully")
logger.Warn("Slow storage detected")
logger.WithError(err).Error("Failed to start VM")
```

---

## Testing

### Unit Tests

```go
package qemu_test

import (
    "context"
    "testing"
    
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    
    "github.com/go-proxmox/gpve/internal/qemu"
)

func TestVMConfig_Validate(t *testing.T) {
    tests := []struct {
        name    string
        config  qemu.VMConfig
        wantErr bool
        errMsg  string
    }{
        {
            name: "valid config",
            config: qemu.VMConfig{
                VMID:   100,
                Cores:  4,
                Memory: 4096,
            },
            wantErr: false,
        },
        {
            name: "invalid vmid",
            config: qemu.VMConfig{
                VMID:   50,
                Cores:  4,
                Memory: 4096,
            },
            wantErr: true,
            errMsg:  "vmid must be >= 100",
        },
        {
            name: "invalid cores",
            config: qemu.VMConfig{
                VMID:   100,
                Cores:  0,
                Memory: 4096,
            },
            wantErr: true,
            errMsg:  "cores must be between 1 and",
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := tt.config.Validate()
            if tt.wantErr {
                require.Error(t, err)
                assert.Contains(t, err.Error(), tt.errMsg)
            } else {
                require.NoError(t, err)
            }
        })
    }
}
```

### Mock Generation

```go
//go:generate mockgen -destination=mocks/mock_storage.go -package=mocks . StorageDriver

// StorageDriver interface for mock generation
type StorageDriver interface {
    Allocate(ctx context.Context, size uint64) (string, error)
    Delete(ctx context.Context, id string) error
}
```

Using mocks in tests:

```go
func TestManager_Create(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()
    
    mockStorage := mocks.NewMockStorageDriver(ctrl)
    mockStorage.EXPECT().
        Allocate(gomock.Any(), uint64(32*1024*1024*1024)).
        Return("vol-123", nil)
    
    mgr := qemu.NewManager(mockStorage)
    
    vm, err := mgr.Create(context.Background(), &qemu.VMConfig{
        VMID: 100,
        Disks: []qemu.DiskConfig{{Size: "32G"}},
    })
    
    require.NoError(t, err)
    assert.Equal(t, uint64(100), vm.VMID)
}
```

### Integration Tests

```go
//go:build integration

package integration_test

import (
    "context"
    "os"
    "testing"
    "time"
    
    "github.com/go-proxmox/gpve/pkg/client"
)

func TestVMLifecycle(t *testing.T) {
    if os.Getenv("GPVE_TEST_HOST") == "" {
        t.Skip("Skipping integration test: GPVE_TEST_HOST not set")
    }
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
    defer cancel()
    
    c, err := client.New(os.Getenv("GPVE_TEST_HOST"), client.WithToken(os.Getenv("GPVE_TEST_TOKEN")))
    require.NoError(t, err)
    
    // Create VM
    vm, err := c.VMs.Create(ctx, &client.VMCreateRequest{
        Name:   "integration-test",
        Cores:  2,
        Memory: 2048,
    })
    require.NoError(t, err)
    t.Cleanup(func() {
        c.VMs.Delete(context.Background(), vm.VMID, true)
    })
    
    // Start VM
    err = c.VMs.Start(ctx, vm.VMID)
    require.NoError(t, err)
    
    // Wait for running
    err = c.VMs.WaitForStatus(ctx, vm.VMID, "running", 60*time.Second)
    require.NoError(t, err)
    
    // Stop VM
    err = c.VMs.Stop(ctx, vm.VMID)
    require.NoError(t, err)
}
```

### E2E Tests

```bash
# Run e2e tests
make test-e2e

# Run specific e2e test
go test -v ./test/e2e/... -run TestClusterFormation
```

---

## Debugging

### Debug Build

```bash
# Build with debug symbols
make build-debug

# Run with delve
dlv exec ./bin/gpve-server -- --config /etc/gpve/gpve.yaml
```

### Debug Logging

```bash
# Enable debug logging
GPVE_LOG_LEVEL=debug ./bin/gpve-server

# Enable component-specific debug
GPVE_DEBUG=qemu,storage ./bin/gpve-server
```

### QEMU Debugging

```bash
# Connect to QMP socket
socat - UNIX-CONNECT:/var/run/gpve/qemu/100.qmp

# Send QMP command
{"execute": "qmp_capabilities"}
{"execute": "query-status"}
{"execute": "query-cpus"}
```

### Profiling

```go
import _ "net/http/pprof"

// Access profiles at:
// http://localhost:6060/debug/pprof/
```

```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Memory profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

---

## Contributing

### Workflow

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Make changes and add tests
4. Run checks: `make check`
5. Commit with conventional commits: `git commit -m "feat: add new feature"`
6. Push and create PR

### Commit Messages

We use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types:

* `feat`: New feature
* `fix`: Bug fix
* `docs`: Documentation
* `style`: Formatting
* `refactor`: Code restructuring
* `perf`: Performance improvement
* `test`: Adding tests
* `chore`: Maintenance

Examples:

```
feat(qemu): add live migration support

Implement QEMU live migration using pre-copy algorithm.
Supports both shared and local storage migrations.

Closes #123
```

```
fix(api): handle concurrent VM modifications

Add optimistic locking using digest field to prevent
race conditions when multiple clients modify the same VM.

Fixes #456
```

### Pull Request Checklist

* [ ] Tests added/updated
* [ ] Documentation updated
* [ ] Changelog entry added
* [ ] Linting passes: `make lint`
* [ ] All tests pass: `make test`
* [ ] Commits are squashed/rebased
* [ ] Branch is up to date with main

### Code Review

All PRs require:

* At least 1 approval from maintainer
* All CI checks passing
* No unresolved conversations

---

## Release Process

### Version Scheme

We use [Semantic Versioning](https://semver.org/):

* MAJOR: Breaking API changes
* MINOR: New features, backward compatible
* PATCH: Bug fixes, backward compatible

### Creating a Release

```bash
# Update changelog
make changelog

# Create release tag
git tag -a v1.2.3 -m "Release v1.2.3"
git push origin v1.2.3

# GoReleaser handles the rest via GitHub Actions
```

### Release Artifacts

* Binary archives (tar.gz, zip)
* Docker images
* DEB/RPM packages
* Checksums (SHA256)
* Signatures (GPG)

---

## Architecture Decisions

### ADR Template

```markdown
# ADR-001: Use etcd for Cluster State

## Status
Accepted

## Context
We need a distributed data store for cluster state that provides
strong consistency and is well-tested in production environments.

## Decision
We will use etcd as the primary cluster state store.

## Consequences
### Positive
- Proven technology (powers Kubernetes)
- Strong consistency via Raft
- Watch support for real-time updates
- Good Go client library

### Negative
- Additional dependency
- Requires 3+ nodes for HA
- Memory overhead for large clusters

## Alternatives Considered
- Consul: More features but more complex
- SQLite + Raft: Custom implementation risk
- CockroachDB: Overkill for our use case
```

### Key ADRs

| ADR     | Title                         | Status   |
| ------- | ----------------------------- | -------- |
| ADR-001 | Use etcd for Cluster State    | Accepted |
| ADR-002 | Proxmox API Compatibility     | Accepted |
| ADR-003 | QEMU as Primary Hypervisor    | Accepted |
| ADR-004 | Go as Implementation Language | Accepted |
| ADR-005 | vCluster for Lightweight K8s  | Accepted |