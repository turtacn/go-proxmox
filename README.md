<div align="center">

  <img src="logo.png" alt="go-proxmox Logo" width="200" height="200">
  
  # go-proxmox
  
  **The Cloud-Native Unified Compute Platform for VMs and Virtual Kubernetes Clusters**
  
  [![Build Status](https://img.shields.io/github/actions/workflow/status/turtacn/go-proxmox/ci.yml?branch=main&style=flat-square)](https://github.com/turtacn/go-proxmox/actions)
  [![Go Version](https://img.shields.io/github/go-mod/go-version/turtacn/go-proxmox?style=flat-square)](https://go.dev/)
  [![License](https://img.shields.io/badge/license-EPL--2.0-blue?style=flat-square)](LICENSE)
  [![Go Report Card](https://goreportcard.com/badge/github.com/turtacn/go-proxmox?style=flat-square)](https://goreportcard.com/report/github.com/turtacn/go-proxmox)
  [![Documentation](https://img.shields.io/badge/docs-architecture-green?style=flat-square)](docs/architecture.md)
  [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](CONTRIBUTING.md)
  
  [English](README.md) | [中文](README-zh.md)
  
</div>

---

## Mission Statement

**go-proxmox** is a modern, Golang-based reimplementation of Proxmox VE's core virtualization capabilities, designed to unify traditional Virtual Machines (VMs) and lightweight Virtual Kubernetes Clusters (vCluster/VC) as first-class compute resources within a single, coherent management plane.

We preserve the battle-tested philosophy of Proxmox VE—**direct hardware control, minimal dependencies, no libvirt abstraction**—while addressing the emerging demands of cloud-native workloads, multi-tenant isolation, and AI/ML infrastructure.

---

## Why go-proxmox?

### Industry Pain Points We Address

| Pain Point | Traditional Approach | go-proxmox Solution |
|------------|---------------------|---------------------|
| **VM-Container Dichotomy** | Separate management planes, inconsistent APIs | Unified resource model with shared storage/network abstractions |
| **Kubernetes Complexity** | Heavy overhead for small/medium workloads | Native vCluster with embedded SQLite, sub-minute provisioning |
| **Multi-Tenancy Gaps** | Namespace-only isolation, shared control plane | Per-tenant independent apiserver, hardware-level data plane isolation |
| **AI Workload Support** | Manual GPU passthrough, no native scheduling | First-class GPU/TPU resources, AI-optimized placement strategies |
| **Operational Burden** | Multiple binaries, complex configuration | Single binary deployment, declarative YAML configuration |
| **Vendor Lock-in** | Proprietary APIs, opaque implementations | Open source, Proxmox-compatible REST API |

### Key Differentiators

```

┌─────────────────────────────────────────────────────────────────────────────┐
│                         go-proxmox Architecture                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐  │
│  │   VirtualMachine │    │  VirtualCluster  │    │   AI/GPU Workloads     │  │
│  │   (QEMU/KVM)     │    │  (vCluster VC)   │    │   (Passthrough/vGPU)   │  │
│  └────────┬────────┘    └────────┬────────┘    └───────────┬─────────────┘  │
│           │                      │                         │                 │
│           └──────────────────────┼─────────────────────────┘                 │
│                                  ▼                                           │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    Unified Resource Manager                            │  │
│  │     • Shared State Machine  • Common Quota Model  • Unified Events     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│           │                      │                         │                 │
│           ▼                      ▼                         ▼                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐  │
│  │ Storage Plugins  │    │ Network Plugins  │    │   Scheduler Engine     │  │
│  │ ZFS/LVM/Ceph/NFS │    │ Bridge/VLAN/SDN  │    │   HA/Utilization/GPU   │  │
│  └─────────────────┘    └─────────────────┘    └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘

````

---

## Key Features

### Compute Resources

- **Virtual Machines (VM)**
  - Direct QMP control over QEMU/KVM (no libvirt)
  - Live/cold migration with shared/local storage
  - Full lifecycle: create, snapshot, clone, GPU passthrough
  - OVMF/SeaBIOS, Secure Boot, vTPM support

- **Virtual Clusters (VC)** — *First-Class Tenant Resource*
  - Independent Kubernetes control plane per tenant
  - Tiered storage: embedded SQLite (dev) → external etcd (prod)
  - Private Nodes mode with kubelet-in-netns for full conformance
  - Runtime isolation: L1 (runc) → L2 (Kata) → L3 (QEMU microvm)

### Unified Infrastructure

- **Storage Abstraction**
  - Single plugin interface for all content types
  - Backends: Local, ZFS, LVM-thin, Ceph RBD, NFS, iSCSI
  - Shared by VM disks, VC PersistentVolumes, backups, ISOs

- **Network Abstraction**
  - Linux Bridge with VLAN/VXLAN support
  - VM NICs and VC Pod networks on same infrastructure
  - Cluster-wide nftables firewall synchronization

### Enterprise Capabilities

- **Multi-Tenancy**: RBAC, quotas (requests/limits/overcommit), audit logging
- **High Availability**: Automatic failover, fencing, maintenance mode (cordon/drain)
- **Observability**: Prometheus metrics, OpenTelemetry tracing, structured logging
- **Security**: Secure Boot, TPM, immutable audit logs, nftables policies

---

## Performance Benchmarks

go-proxmox is designed for production-grade performance:

| Benchmark | Target | Notes |
|-----------|--------|-------|
| VMs per Node | ≥500 | Standard configurations |
| VCs per Node | ≥1000 | Small-spec virtual clusters |
| Control Plane p99 Latency | <100ms | API operations |
| VC Provisioning | <60s | Including control plane bootstrap |
| Live Migration Downtime | <500ms | Shared storage, standard VM |
| Oracle RAC Certification | Planned | Enterprise database workloads |
| Kubernetes Conformance | 100% | CNCF certified distribution base |

---

## Getting Started

### Prerequisites

- Linux kernel ≥5.15 with KVM support
- Go 1.22+ (for building from source)
- QEMU 8.0+ installed on target nodes

### Installation

**From Binary Releases:**

```bash
# Download latest release
curl -LO https://github.com/turtacn/go-proxmox/releases/latest/download/gpve-linux-amd64.tar.gz
tar xzf gpve-linux-amd64.tar.gz
sudo mv gpve-server gpve-agent /usr/local/bin/

# Initialize configuration
gpve-server init --config /etc/gpve/config.yaml
````

**From Source:**

```bash
# Clone repository
git clone https://github.com/turtacn/go-proxmox.git
cd go-proxmox

# Build all binaries
make build

# Run tests
make test

# Install
sudo make install
```

**Using Go Install:**

```bash
go install github.com/turtacn/go-proxmox/cmd/gpve-server@latest
go install github.com/turtacn/go-proxmox/cmd/gpve-agent@latest
go install github.com/turtacn/go-proxmox/cmd/gpvectl@latest
```

### Quick Start

**1. Start the Server:**

```bash
# Initialize a new cluster
gpve-server init --cluster-name my-cluster

# Start server (foreground for testing)
gpve-server run --config /etc/gpve/config.yaml
```

**2. Start Node Agent:**

```bash
# On each compute node
gpve-agent join --server https://control-plane:8443 --token <bootstrap-token>
```

**3. Create Your First VM:**

```bash
# Using CLI
gpvectl vm create my-vm \
  --cpu 4 \
  --memory 8Gi \
  --disk size=100Gi,storage=local-zfs \
  --network bridge=vmbr0

# Using declarative YAML
cat <<EOF | gpvectl apply -f -
apiVersion: gpve.io/v1
kind: VirtualMachine
metadata:
  name: my-vm
  namespace: default
spec:
  cpu:
    cores: 4
  memory:
    size: 8Gi
  disks:
    - name: root
      size: 100Gi
      storage: local-zfs
  networks:
    - name: eth0
      bridge: vmbr0
EOF
```

**4. Create a Virtual Cluster:**

```bash
gpvectl vc create my-cluster \
  --k8s-version 1.30 \
  --control-plane-tier standard \
  --worker-nodes 3 \
  --isolation-level L2

# Get kubeconfig
gpvectl vc kubeconfig my-cluster > ~/.kube/my-cluster.yaml
export KUBECONFIG=~/.kube/my-cluster.yaml
kubectl get nodes
```

---

## Usage Examples

### VM Lifecycle Management

```go
package main

import (
    "context"
    "log"

    "github.com/turtacn/go-proxmox/pkg/client"
    "github.com/turtacn/go-proxmox/pkg/api/v1"
)

func main() {
    // Create client
    c, err := client.New("https://gpve-server:8443", client.WithToken("your-token"))
    if err != nil {
        log.Fatal(err)
    }

    // Define VM specification
    vm := &v1.VirtualMachine{
        ObjectMeta: v1.ObjectMeta{
            Name:      "web-server",
            Namespace: "production",
        },
        Spec: v1.VirtualMachineSpec{
            CPU: v1.CPUSpec{
                Cores:   4,
                Sockets: 1,
                Type:    "host",
            },
            Memory: v1.MemorySpec{
                Size: "16Gi",
            },
            Disks: []v1.DiskSpec{
                {
                    Name:    "root",
                    Size:    "100Gi",
                    Storage: "ceph-pool",
                    Boot:    true,
                },
            },
            Networks: []v1.NetworkSpec{
                {
                    Name:   "eth0",
                    Bridge: "vmbr0",
                    VLAN:   100,
                },
            },
        },
    }

    // Create VM
    created, err := c.VirtualMachines("production").Create(context.Background(), vm)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("VM created: %s (ID: %d)", created.Name, created.Status.VMID)

    // Start VM
    if err := c.VirtualMachines("production").Start(context.Background(), "web-server"); err != nil {
        log.Fatal(err)
    }

    // Live migrate to another node
    if err := c.VirtualMachines("production").Migrate(context.Background(), "web-server", 
        client.MigrateOptions{
            TargetNode: "node-02",
            Online:     true,
        }); err != nil {
        log.Fatal(err)
    }
}
```

### Virtual Cluster Operations

```go
package main

import (
    "context"
    "log"

    "github.com/turtacn/go-proxmox/pkg/client"
    "github.com/turtacn/go-proxmox/pkg/api/v1"
)

func main() {
    c, _ := client.New("https://gpve-server:8443", client.WithToken("your-token"))

    // Create a virtual cluster for tenant
    vc := &v1.VirtualCluster{
        ObjectMeta: v1.ObjectMeta{
            Name:      "tenant-alpha-cluster",
            Namespace: "tenants",
            Labels: map[string]string{
                "tenant": "alpha",
                "env":    "production",
            },
        },
        Spec: v1.VirtualClusterSpec{
            KubernetesVersion: "1.30.2",
            ControlPlane: v1.ControlPlaneSpec{
                Tier:         v1.ControlPlaneTierPerformance,
                BackingStore: v1.BackingStoreEtcd,
                Replicas:     3,
            },
            WorkerNodes: v1.WorkerNodesSpec{
                Mode:  v1.WorkerModePrivate,
                Count: 5,
                Resources: v1.ResourceRequirements{
                    Requests: v1.ResourceList{
                        v1.ResourceCPU:    "2",
                        v1.ResourceMemory: "4Gi",
                    },
                    Limits: v1.ResourceList{
                        v1.ResourceCPU:    "4",
                        v1.ResourceMemory: "8Gi",
                    },
                },
            },
            IsolationLevel: v1.IsolationLevelL2, // Kata Containers
            Scheduling: v1.SchedulingSpec{
                Strategy: v1.SchedulingStrategyHA,
                AntiAffinity: &v1.AntiAffinitySpec{
                    TopologyKey: "kubernetes.io/hostname",
                },
            },
        },
    }

    created, err := c.VirtualClusters("tenants").Create(context.Background(), vc)
    if err != nil {
        log.Fatal(err)
    }

    // Wait for cluster to be ready
    c.VirtualClusters("tenants").WaitForReady(context.Background(), created.Name)

    // Get kubeconfig
    kubeconfig, err := c.VirtualClusters("tenants").GetKubeconfig(context.Background(), created.Name)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("Kubeconfig:\n%s", kubeconfig)
}
```

### Storage Plugin Example

```go
package main

import (
    "context"
    
    "github.com/turtacn/go-proxmox/pkg/storage"
    "github.com/turtacn/go-proxmox/pkg/storage/plugins/zfs"
)

func main() {
    // Initialize ZFS storage backend
    zfsBackend, err := zfs.New(zfs.Config{
        PoolName:   "tank",
        Dataset:    "gpve",
        Mountpoint: "/tank/gpve",
    })
    if err != nil {
        panic(err)
    }

    // Register with storage manager
    mgr := storage.NewManager()
    mgr.RegisterPlugin("local-zfs", zfsBackend)

    // Create a volume for VM
    vol, err := mgr.CreateVolume(context.Background(), "local-zfs", storage.VolumeSpec{
        Name:        "vm-100-disk-0",
        Size:        100 * 1024 * 1024 * 1024, // 100GiB
        ContentType: storage.ContentTypeImage,
        Format:      storage.FormatRaw,
    })
    if err != nil {
        panic(err)
    }

    // Create snapshot
    snap, err := mgr.CreateSnapshot(context.Background(), vol.ID, "before-upgrade")
    if err != nil {
        panic(err)
    }
    
    _ = snap
}
```

---

## Architecture Overview

For detailed architecture documentation, see [docs/architecture.md](docs/architecture.md).

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              go-proxmox Architecture                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                         Presentation Layer                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │   REST API    │  │   gRPC API   │  │   CLI Tool   │  │  Web UI (*)  │   │ │
│  │  │  (Proxmox     │  │  (Internal   │  │  (gpvectl)   │  │  (Future)    │   │ │
│  │  │   Compatible) │  │   Cluster)   │  │              │  │              │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                          │
│  ┌────────────────────────────────────▼───────────────────────────────────────┐ │
│  │                         Application Layer                                   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │ VM Service    │  │ VC Service   │  │ Storage Svc  │  │ Network Svc  │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │ Cluster Svc   │  │ HA Service   │  │ Task Service │  │ Auth Service │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                          │
│  ┌────────────────────────────────────▼───────────────────────────────────────┐ │
│  │                           Domain Layer                                      │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │ VM Domain     │  │ VC Domain    │  │ Storage Dom  │  │ Network Dom  │   │ │
│  │  │ (QMP/QEMU)    │  │ (vCluster)   │  │ (Plugins)    │  │ (Bridge/SDN) │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │ │
│  │  │ Cluster Dom   │  │ Scheduler    │  │ State Machine│                     │ │
│  │  │ (Consensus)   │  │ (Placement)  │  │ (Lifecycle)  │                     │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                     │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                          │
│  ┌────────────────────────────────────▼───────────────────────────────────────┐ │
│  │                       Infrastructure Layer                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │ QMP Client    │  │ Storage APIs │  │ Network APIs │  │ Cluster Store│   │ │
│  │  │ (QEMU)        │  │ (ZFS/LVM/...)│  │ (netlink)    │  │ (etcd/CRDT)  │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │ Metrics       │  │ Logging      │  │ Tracing      │  │ Audit Log    │   │ │
│  │  │ (Prometheus)  │  │ (Structured) │  │ (OTel)       │  │ (Immutable)  │   │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## API Documentation

Full API documentation is available at [docs/apis.md](docs/apis.md).

**API Compatibility:**

* REST API paths compatible with Proxmox VE (VM operations)
* Extended paths for vCluster resources
* gRPC for internal cluster communication

**Example Endpoints:**

```
# VM Operations (Proxmox-compatible)
GET    /api2/json/nodes/{node}/qemu
POST   /api2/json/nodes/{node}/qemu
GET    /api2/json/nodes/{node}/qemu/{vmid}/status/current
POST   /api2/json/nodes/{node}/qemu/{vmid}/status/start
POST   /api2/json/nodes/{node}/qemu/{vmid}/migrate

# VirtualCluster Operations (Extended)
GET    /api/v1/tenants/{tenantId}/vclusters
POST   /api/v1/tenants/{tenantId}/vclusters
GET    /api/v1/tenants/{tenantId}/vclusters/{name}
GET    /api/v1/tenants/{tenantId}/vclusters/{name}/kubeconfig
POST   /api/v1/tenants/{tenantId}/vclusters/{name}/upgrade
```

---

## Roadmap

| Phase   | Status         | Description                                    |
| ------- | -------------- | ---------------------------------------------- |
| Phase 0 | ✅ Complete     | Project scaffold, core types, build system     |
| Phase 1 | 🚧 In Progress | Storage abstraction, basic VM lifecycle        |
| Phase 2 | 📋 Planned     | Network abstraction, QMP integration           |
| Phase 3 | 📋 Planned     | Cluster consensus, node management             |
| Phase 4 | 📋 Planned     | VirtualCluster implementation                  |
| Phase 5 | 📋 Planned     | HA, migration, production hardening            |
| Phase 6 | 📋 Planned     | Web UI, advanced SDN, AI workload optimization |

---

## Contributing

We welcome contributions from the community! Please read our [Contributing Guide](CONTRIBUTING.md) before submitting PRs.

### Development Setup

```bash
# Clone the repo
git clone https://github.com/turtacn/go-proxmox.git
cd go-proxmox

# Install development dependencies
make deps

# Run linters
make lint

# Run all tests
make test

# Run integration tests (requires QEMU)
make test-integration

# Build for current platform
make build
```

### Code Style

* Follow standard Go conventions
* Run `make lint` before committing
* Ensure test coverage ≥80% for new code
* Document all exported types and functions

---

## Community

* **GitHub Discussions**: [github.com/turtacn/go-proxmox/discussions](https://github.com/turtacn/go-proxmox/discussions)
* **Issue Tracker**: [github.com/turtacn/go-proxmox/issues](https://github.com/turtacn/go-proxmox/issues)
* **Security Issues**: [security@go-proxmox.dev](mailto:security@go-proxmox.dev) (PGP key available)

---

## License

go-proxmox is licensed under the [Eclipse Public License 2.0](LICENSE).

```
Copyright (c) 2024-2025 go-proxmox Contributors

This program and the accompanying materials are made available under the
terms of the Eclipse Public License 2.0 which is available at
http://www.eclipse.org/legal/epl-2.0
```

---

## Acknowledgments

* [Proxmox VE](https://www.proxmox.com/) for the foundational virtualization concepts
* [vCluster](https://www.vcluster.com/) for virtual Kubernetes cluster technology
* [K3s](https://k3s.io/) for lightweight Kubernetes distribution
* [QEMU](https://www.qemu.org/) for virtualization backend

---

<div align="center">
  <sub>Built with ❤️ by the go-proxmox community</sub>
</div>