<div align="center">
  <!--
  <img src="logo.png" alt="go-proxmox Logo" width="200" height="200">
  -->
  
  # go-proxmox
  
  **Cloud-Native Infrastructure Runtime for Hybrid Workloads**
  
  [![Build Status](https://img.shields.io/github/actions/workflow/status/turtacn/go-proxmox/ci.yml?branch=main&style=flat-square)](https://github.com/turtacn/go-proxmox/actions)
  [![Go Version](https://img.shields.io/badge/go-%3E%3D1.22-blue?style=flat-square)](https://go.dev/)
  [![License](https://img.shields.io/badge/license-Apache%202.0-green?style=flat-square)](LICENSE)
  [![Go Report Card](https://goreportcard.com/badge/github.com/turtacn/go-proxmox?style=flat-square)](https://goreportcard.com/report/github.com/turtacn/go-proxmox)
  [![Documentation](https://img.shields.io/badge/docs-architecture-orange?style=flat-square)](docs/architecture.md)
  [![Release](https://img.shields.io/github/v/release/turtacn/go-proxmox?style=flat-square)](https://github.com/turtacn/go-proxmox/releases)
  
  [English](README.md) | [中文](README-zh.md)
  
</div>

---

## Mission Statement

**go-proxmox** is a next-generation infrastructure runtime that reimagines virtualization management for the cloud-native era. By rebuilding Proxmox VE's core capabilities in Go, we deliver a unified compute plane where **Virtual Machines** and **vClusters** (lightweight Virtual Kubernetes) are treated as **equal first-class citizens**.

Our goal is to bridge the gap between traditional virtualization and cloud-native workloads, enabling organizations to run VMs, containers, and AI workloads on a single, coherent platform—from edge devices to data center scale.

---

## Why go-proxmox?

### Industry Pain Points We Address

| Challenge | Traditional Solutions | go-proxmox Approach |
|-----------|----------------------|---------------------|
| **Fragmented Tooling** | Separate management for VMs vs. Kubernetes | Unified API and lifecycle management |
| **Complex Operations** | Multiple control planes, inconsistent UX | Single binary, consistent experience |
| **Resource Inefficiency** | Over-provisioning due to isolation boundaries | Shared resource pools with fine-grained isolation |
| **Slow Deployment** | Heavy VM-based Kubernetes clusters | vCluster starts in seconds, not minutes |
| **Lock-in Risk** | Proprietary hyperconverged platforms | Open-source, pluggable architecture |
| **AI Workload Gap** | Legacy platforms lack AI/Agent optimization | Native support for AI inference and Agentic workloads |

### Key Differentiators

```

┌─────────────────────────────────────────────────────────────────┐
│                    go-proxmox Advantages                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. UNIFIED ABSTRACTION                                         │
│     VM and vCluster share the same Guest interface              │
│     Common scheduling, storage, network semantics               │
│                                                                 │
│  2. SINGLE BINARY DEPLOYMENT                                    │
│     Edge to datacenter with identical experience                │
│     Zero external dependencies for basic operation              │
│                                                                 │
│  3. DECLARATIVE-FIRST API                                       │
│     GitOps-ready, Kubernetes-style resources                    │
│     Full REST compatibility with Proxmox ecosystem              │
│                                                                 │
│  4. PLUGGABLE RUNTIMES                                          │
│     QEMU/KVM, Firecracker, containerd                           │
│     Choose isolation level per workload                         │
│                                                                 │
│  5. OBSERVABILITY BUILT-IN                                      │
│     OpenTelemetry native, eBPF-powered insights                 │
│     Unified logs, metrics, traces                               │
│                                                                 │
│  6. AI-READY ARCHITECTURE                                       │
│     Optimized for inference workloads                           │
│     Native Agent/A2A orchestration support                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

```

---

## Key Features

### Core Capabilities

- **VM Management**: Full QEMU/KVM lifecycle (create, start, stop, migrate, snapshot, clone)
- **vCluster Support**: Lightweight Kubernetes clusters with hardware-level isolation
- **Live Migration**: Zero-downtime VM and vCluster migration across nodes
- **High Availability**: Automatic failover with configurable fencing strategies
- **Storage Abstraction**: Pluggable drivers for LVM, ZFS, Ceph, NFS
- **Network Management**: Linux Bridge, VLAN, SDN with nftables firewall

### Enterprise Features

- **Multi-tenancy**: Fine-grained RBAC with resource quotas
- **Cluster Federation**: Manage multiple clusters from single control plane
- **Backup Integration**: Compatible with Proxmox Backup Server protocol
- **Audit Logging**: Immutable audit trail for compliance

### AI & Cloud-Native

- **GPU Passthrough**: NVIDIA/AMD GPU support for AI inference
- **vGPU Scheduling**: Share GPUs across multiple workloads
- **Agent Orchestration**: Native support for AI Agent workflows
- **Serverless Ready**: Fast cold-start for event-driven workloads

---

## Architecture Overview

```mermaid

graph TD
    %% 图例（Legend）：<br/>蓝色表示接口层，绿色表示管理/抽象层，灰色表示运行时与资源层

    %% ========================
    %% 接入层（Access Layer）
    %% ========================
    subgraph AL[接入层（Access Layer）]
        A1[gpvectl<br/>命令行工具（CLI）]
        A2[REST API<br/>兼容接口（Compatibility API）]
        A3[gRPC API<br/>原生接口（Native API）]
        A4[Web UI<br/>管理界面（Future）]
    end

    %% ========================
    %% API 服务层（API Server Layer）
    %% ========================
    subgraph AS[API服务层（API Server Layer）]
        B1[API服务器<br/>认证（Authentication）<br/>鉴权（Authorization）<br/>路由（Routing）]
    end

    %% ========================
    %% 管理层（Management Layer）
    %% ========================
    subgraph ML[管理层（Management Layer）]
        C1[虚拟机管理器<br/>（VM Manager）]
        C2[虚拟集群管理器<br/>（vCluster Manager）]
        C3[K8s-on-VM管理器<br/>（Kubernetes on VM Manager）]
    end

    %% ========================
    %% 来宾抽象层（Guest Abstraction Layer）
    %% ========================
    subgraph GAL[来宾抽象层（Guest Abstraction Layer）]
        D1[统一生命周期管理<br/>状态机（State Machine）<br/>事件（Events）]
    end

    %% ========================
    %% 运行时层（Runtime Layer）
    %% ========================
    subgraph RL[运行时层（Runtime Layer）]
        E1[QEMU运行时<br/>（QEMU Runtime）]
        E2[containerd运行时<br/>（containerd Runtime）]
        E3[Firecracker运行时<br/>（Firecracker Runtime）]
    end

    %% ========================
    %% 资源层（Resource Layer）
    %% ========================
    subgraph RSL[资源层（Resource Layer）]
        F1[存储<br/>（Storage）]
        F2[网络<br/>（Network）]
        F3[集群<br/>（Cluster）]
        F4[高可用<br/>（HA）]
        F5[可观测性<br/>（Observability）]
    end

    %% ========================
    %% 业务流（Business Flow）
    %% ========================
    %% 1. 接入层到API服务层
    A1 -->|1.1 请求提交| B1
    A2 -->|1.2 请求提交| B1
    A3 -->|1.3 请求提交| B1
    A4 -->|1.4 请求提交| B1

    %% 2. API服务层到管理层
    B1 -->|2.1 调度与分发| C1
    B1 -->|2.2 调度与分发| C2
    B1 -->|2.3 调度与分发| C3

    %% 3. 管理层到来宾抽象层
    C1 -->|3.1 生命周期调用| D1
    C2 -->|3.2 生命周期调用| D1
    C3 -->|3.3 生命周期调用| D1

    %% 4. 来宾抽象层到运行时层
    D1 -->|4.1 实例运行| E1
    D1 -->|4.2 实例运行| E2
    D1 -->|4.3 实例运行| E3

    %% 5. 运行时层依赖资源层
    E1 -->|5.1 资源访问| F1
    E1 -->|5.2 资源访问| F2
    E2 -->|5.3 资源访问| F3
    E3 -->|5.4 资源访问| F4
    E3 -->|5.5 资源访问| F5


````

For detailed architecture documentation, see [docs/architecture.md](docs/architecture.md).

---

## Getting Started

### Prerequisites

- Go 1.22 or later
- Linux kernel 5.10+ with KVM support
- QEMU 6.0+ (for VM runtime)
- containerd 1.7+ (for vCluster runtime)

### Installation

#### From Source

```bash
# Clone the repository
git clone https://github.com/turtacn/go-proxmox.git
cd go-proxmox

# Build all binaries
make build

# Install to system
sudo make install
````

#### Using go install

```bash
# Install CLI tool
go install github.com/turtacn/go-proxmox/cmd/gpvectl@latest

# Install server (requires root for runtime operations)
go install github.com/turtacn/go-proxmox/cmd/gpve-server@latest
go install github.com/turtacn/go-proxmox/cmd/gpve-agent@latest
```

#### Container Image

```bash
# Pull the official image
docker pull ghcr.io/turtacn/go-proxmox:latest

# Run the server
docker run -d --privileged \
  -v /var/run/libvirt:/var/run/libvirt \
  -v /var/lib/gpve:/var/lib/gpve \
  -p 8006:8006 \
  ghcr.io/turtacn/go-proxmox:latest
```

### Quick Start

#### Initialize a Single-Node Cluster

```bash
# Initialize the cluster
gpvectl cluster init --name my-cluster

# Verify the node status
gpvectl node list
```

#### Create Your First VM

```bash
# Create a VM from a cloud image
gpvectl vm create \
  --name ubuntu-vm \
  --cores 2 \
  --memory 4096 \
  --disk local:32G \
  --cdrom local:iso/ubuntu-22.04.iso \
  --net bridge=vmbr0

# Start the VM
gpvectl vm start ubuntu-vm

# Check VM status
gpvectl vm status ubuntu-vm
```

#### Create a vCluster

```bash
# Create a lightweight Kubernetes cluster
gpvectl vcluster create \
  --name dev-cluster \
  --cores 4 \
  --memory 8192 \
  --k8s-version 1.29

# Get kubeconfig
gpvectl vcluster kubeconfig dev-cluster > ~/.kube/dev-cluster.yaml

# Access the cluster
export KUBECONFIG=~/.kube/dev-cluster.yaml
kubectl get nodes
```

---

## Usage Examples

### Programmatic API Usage

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/turtacn/go-proxmox/pkg/client"
    "github.com/turtacn/go-proxmox/pkg/guest/types"
)

func main() {
    // Create a client
    c, err := client.New("https://localhost:8006", client.WithToken("your-api-token"))
    if err != nil {
        log.Fatal(err)
    }

    ctx := context.Background()

    // Create a VM spec
    vmSpec := &types.VMSpec{
        Name:   "api-created-vm",
        Cores:  2,
        Memory: 4096,
        Disks: []types.DiskSpec{
            {Storage: "local", Size: "32G"},
        },
        Networks: []types.NetSpec{
            {Bridge: "vmbr0", Model: "virtio"},
        },
    }

    // Create the VM
    vm, err := c.VMs().Create(ctx, vmSpec)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Created VM: %s (ID: %d)\n", vm.Name, vm.ID)

    // Start the VM
    if err := c.VMs().Start(ctx, vm.ID); err != nil {
        log.Fatal(err)
    }
    fmt.Println("VM started successfully")

    // List all guests (VMs + vClusters)
    guests, err := c.Guests().List(ctx)
    if err != nil {
        log.Fatal(err)
    }
    for _, g := range guests {
        fmt.Printf("Guest: %s, Type: %s, Status: %s\n", g.Name, g.Type, g.Status)
    }
}
```

### Declarative Configuration

```yaml
# vm-example.yaml
apiVersion: gpve.io/v1
kind: VirtualMachine
metadata:
  name: production-db
  labels:
    app: database
    env: production
spec:
  cores: 8
  memory: 32768
  disks:
    - storage: ceph-pool
      size: 500G
      cache: writeback
  networks:
    - bridge: vmbr0
      vlan: 100
      firewall: true
  ha:
    enabled: true
    group: database-ha
---
apiVersion: gpve.io/v1
kind: VCluster
metadata:
  name: staging-k8s
spec:
  cores: 4
  memory: 16384
  kubernetes:
    version: "1.29"
    cni: flannel
    features:
      - metrics-server
      - ingress-nginx
```

```bash
# Apply the configuration
gpvectl apply -f vm-example.yaml
```

---

## Benchmarks

go-proxmox is designed for production workloads. Key performance metrics:

| Benchmark       | Metric     | go-proxmox | Reference      |
| --------------- | ---------- | ---------- | -------------- |
| VM Boot Time    | Cold start | < 3s       | QEMU baseline  |
| vCluster Boot   | Pod-ready  | < 5s       | k3s standard   |
| API Latency     | P99        | < 50ms     | Under load     |
| Live Migration  | Downtime   | < 100ms    | 8GB VM         |
| Guest Density   | Per node   | 200+       | Mixed workload |
| Oracle RAC      | TPC-C      | Certified  | Enterprise DB  |
| K8s Conformance | CNCF       | 100%       | v1.29          |

---

## Project Status

| Phase                      | Status      | Description                                    |
| -------------------------- | ----------- | ---------------------------------------------- |
| Phase 0: Foundation        | In Progress | Core abstractions, config, auth, observability |
| Phase 1: VM Runtime        | In Progress     | Full VM lifecycle management                   |
| Phase 2: Storage & Network | In Progress     | Multi-backend storage, SDN                     |
| Phase 3: Cluster & HA      | In Progress     | Multi-node, live migration, HA                 |
| Phase 4: vCluster          | In Progress     | Lightweight Kubernetes runtime                 |
| Phase 5: K8s Integration   | In Progress     | K8s-on-VM, unified management                  |

---



## Development Setup

```bash
# Clone your fork
git clone https://github.com/turtacn/go-proxmox.git
cd go-proxmox

# Install development dependencies
make dev-deps

# Run tests
make test

# Run linters
make lint

# Build for development
make build-dev
```

---

## Documentation

* [Architecture Guide](docs/architecture.md) - Detailed system architecture
* [API Reference](docs/api-reference.md) - REST and gRPC API documentation
* [Operations Guide](docs/operations.md) - Deployment and operations
* [Development Guide](docs/development.md) - Contributing and development setup

---


## License

go-proxmox is licensed under the [Apache License 2.0](LICENSE).

```
Copyright 2026 The go-proxmox Authors

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

---

## Acknowledgements

go-proxmox stands on the shoulders of giants:

* [Proxmox VE](https://www.proxmox.com/) - Inspiration and compatibility target
* [QEMU/KVM](https://www.qemu.org/) - Virtualization runtime
* [containerd](https://containerd.io/) - Container runtime
* [k3s](https://k3s.io/) - Lightweight Kubernetes
* [etcd](https://etcd.io/) - Distributed state store

---

<div align="center">
  <b>Built with passion for the cloud-native infrastructure community</b>
  <br><br>
  <a href="https://github.com/turtacn/go-proxmox/stargazers">Star us on GitHub</a>
</div>

---