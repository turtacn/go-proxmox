# API Reference

## Overview

go-proxmox provides two API interfaces:
1. **Proxmox-Compatible REST API** - For compatibility with existing tools
2. **Native REST API** - Extended functionality with modern design
3. **gRPC API** - High-performance programmatic access

Base URL: `https://<host>:8006`

## Authentication

### Obtain Access Ticket

```http
POST /api2/json/access/ticket
Content-Type: application/x-www-form-urlencoded

username=root@pam&password=secret
````

**Response:**

```json
{
  "data": {
    "ticket": "PVE:root@pam:6789ABCD::BASE64_SIGNATURE",
    "CSRFPreventionToken": "6789ABCD:BASE64_TOKEN",
    "username": "root@pam",
    "cap": {
      "vms": { "VM.Allocate": 1, "VM.Clone": 1 },
      "storage": { "Datastore.Allocate": 1 },
      "nodes": { "Sys.Audit": 1 }
    }
  }
}
```

### Using the Ticket

Include in subsequent requests:

```http
GET /api2/json/nodes
Cookie: PVEAuthCookie=PVE:root@pam:6789ABCD::BASE64_SIGNATURE
CSRFPreventionToken: 6789ABCD:BASE64_TOKEN
```

### API Tokens

Create long-lived API tokens:

```http
POST /api2/json/access/users/{userid}/token/{tokenid}
Content-Type: application/json

{
  "privsep": true,
  "expire": 0,
  "comment": "Terraform automation"
}
```

Use token in requests:

```http
GET /api2/json/nodes
Authorization: PVEAPIToken=user@realm!tokenid=TOKEN_SECRET
```

---

## Nodes

### List Nodes

```http
GET /api2/json/nodes
```

**Response:**

```json
{
  "data": [
    {
      "node": "node1",
      "status": "online",
      "cpu": 0.15,
      "maxcpu": 16,
      "mem": 17179869184,
      "maxmem": 68719476736,
      "disk": 107374182400,
      "maxdisk": 1099511627776,
      "uptime": 864000,
      "level": ""
    }
  ]
}
```

### Get Node Status

```http
GET /api2/json/nodes/{node}/status
```

**Response:**

```json
{
  "data": {
    "uptime": 864000,
    "kversion": "6.5.0-1-amd64",
    "cpuinfo": {
      "model": "AMD EPYC 7763",
      "cores": 64,
      "sockets": 2,
      "mhz": "2450.000"
    },
    "memory": {
      "total": 68719476736,
      "used": 17179869184,
      "free": 51539607552
    },
    "rootfs": {
      "total": 1099511627776,
      "used": 107374182400,
      "avail": 992137224576
    },
    "loadavg": ["0.50", "0.75", "0.60"]
  }
}
```

---

## Virtual Machines

### List VMs on Node

```http
GET /api2/json/nodes/{node}/qemu
```

**Response:**

```json
{
  "data": [
    {
      "vmid": 100,
      "name": "ubuntu-server",
      "status": "running",
      "mem": 4294967296,
      "maxmem": 8589934592,
      "cpu": 0.05,
      "maxcpu": 4,
      "uptime": 86400,
      "pid": 12345,
      "template": false
    }
  ]
}
```

### Create VM

```http
POST /api2/json/nodes/{node}/qemu
Content-Type: application/json

{
  "vmid": 101,
  "name": "new-vm",
  "memory": 4096,
  "cores": 2,
  "sockets": 1,
  "cpu": "host",
  "ostype": "l26",
  "scsihw": "virtio-scsi-single",
  "scsi0": "local-lvm:32,format=raw",
  "net0": "virtio,bridge=vmbr0",
  "ide2": "local:iso/ubuntu-22.04.iso,media=cdrom",
  "boot": "order=scsi0;ide2",
  "start": false
}
```

**Response:**

```json
{
  "data": "UPID:node1:00001234:12345678:65432100:qmcreate:101:root@pam:"
}
```

### Get VM Configuration

```http
GET /api2/json/nodes/{node}/qemu/{vmid}/config
```

**Response:**

```json
{
  "data": {
    "name": "ubuntu-server",
    "memory": 4096,
    "cores": 2,
    "sockets": 1,
    "cpu": "host",
    "ostype": "l26",
    "scsihw": "virtio-scsi-single",
    "scsi0": "local-lvm:vm-100-disk-0,size=32G",
    "net0": "virtio=AA:BB:CC:DD:EE:FF,bridge=vmbr0",
    "boot": "order=scsi0",
    "digest": "abc123..."
  }
}
```

### Update VM Configuration

```http
PUT /api2/json/nodes/{node}/qemu/{vmid}/config
Content-Type: application/json

{
  "memory": 8192,
  "cores": 4,
  "digest": "abc123..."
}
```

### VM Status Operations

#### Get Current Status

```http
GET /api2/json/nodes/{node}/qemu/{vmid}/status/current
```

#### Start VM

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/status/start
```

#### Stop VM

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/status/stop

{
  "timeout": 60,
  "forceStop": false
}
```

#### Shutdown VM (ACPI)

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/status/shutdown

{
  "timeout": 120,
  "forceStop": true
}
```

#### Reboot VM

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/status/reboot
```

#### Reset VM

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/status/reset
```

#### Suspend VM

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/status/suspend
```

#### Resume VM

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/status/resume
```

### Delete VM

```http
DELETE /api2/json/nodes/{node}/qemu/{vmid}

{
  "purge": true,
  "destroy-unreferenced-disks": true
}
```

### Clone VM

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/clone
Content-Type: application/json

{
  "newid": 102,
  "name": "cloned-vm",
  "target": "node2",
  "full": true,
  "storage": "local-lvm"
}
```

### Migrate VM

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/migrate
Content-Type: application/json

{
  "target": "node2",
  "online": true,
  "with-local-disks": true,
  "targetstorage": "local-lvm"
}
```

### Snapshot Operations

#### List Snapshots

```http
GET /api2/json/nodes/{node}/qemu/{vmid}/snapshot
```

#### Create Snapshot

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/snapshot
Content-Type: application/json

{
  "snapname": "before-upgrade",
  "description": "Snapshot before system upgrade",
  "vmstate": true
}
```

#### Rollback Snapshot

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/snapshot/{snapname}/rollback
```

#### Delete Snapshot

```http
DELETE /api2/json/nodes/{node}/qemu/{vmid}/snapshot/{snapname}
```

### Console Access

#### VNC Proxy

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/vncproxy

{
  "websocket": true
}
```

**Response:**

```json
{
  "data": {
    "port": "5900",
    "ticket": "VNCTICKET",
    "upid": "UPID:...",
    "user": "root@pam"
  }
}
```

#### SPICE Proxy

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/spiceproxy
```

#### Terminal Proxy

```http
POST /api2/json/nodes/{node}/qemu/{vmid}/termproxy
```

---

## Storage

### List Storage

```http
GET /api2/json/nodes/{node}/storage
```

**Response:**

```json
{
  "data": [
    {
      "storage": "local",
      "type": "dir",
      "active": 1,
      "enabled": 1,
      "shared": 0,
      "total": 107374182400,
      "used": 21474836480,
      "avail": 85899345920,
      "content": "images,iso,vztmpl,snippets"
    },
    {
      "storage": "local-lvm",
      "type": "lvmthin",
      "active": 1,
      "enabled": 1,
      "shared": 0,
      "total": 536870912000,
      "used": 107374182400,
      "avail": 429496729600,
      "content": "rootdir,images"
    }
  ]
}
```

### List Storage Content

```http
GET /api2/json/nodes/{node}/storage/{storage}/content

# With filters
GET /api2/json/nodes/{node}/storage/{storage}/content?content=iso
```

**Response:**

```json
{
  "data": [
    {
      "volid": "local:iso/ubuntu-22.04.iso",
      "format": "iso",
      "size": 1474873344,
      "ctime": 1699900000
    },
    {
      "volid": "local-lvm:vm-100-disk-0",
      "format": "raw",
      "size": 34359738368,
      "vmid": 100
    }
  ]
}
```

### Upload ISO/Template

```http
POST /api2/json/nodes/{node}/storage/{storage}/upload
Content-Type: multipart/form-data

content=iso
filename=@/path/to/ubuntu-22.04.iso
```

### Download from URL

```http
POST /api2/json/nodes/{node}/storage/{storage}/download-url
Content-Type: application/json

{
  "content": "iso",
  "filename": "ubuntu-22.04.iso",
  "url": "https://releases.ubuntu.com/22.04/ubuntu-22.04.3-live-server-amd64.iso",
  "checksum": "sha256:...",
  "checksum-algorithm": "sha256"
}
```

### Delete Volume

```http
DELETE /api2/json/nodes/{node}/storage/{storage}/content/{volume}
```

---

## Network

### List Networks

```http
GET /api2/json/nodes/{node}/network
```

**Response:**

```json
{
  "data": [
    {
      "iface": "eth0",
      "type": "eth",
      "active": 1,
      "autostart": 1,
      "method": "manual"
    },
    {
      "iface": "vmbr0",
      "type": "bridge",
      "active": 1,
      "autostart": 1,
      "method": "static",
      "address": "10.0.0.1",
      "netmask": "255.255.255.0",
      "gateway": "10.0.0.254",
      "bridge_ports": "eth0",
      "bridge_stp": "off",
      "bridge_fd": "0"
    }
  ]
}
```

### Create Network

```http
POST /api2/json/nodes/{node}/network
Content-Type: application/json

{
  "iface": "vmbr1",
  "type": "bridge",
  "autostart": true,
  "bridge_ports": "eth1",
  "bridge_vlan_aware": true,
  "address": "192.168.1.1",
  "netmask": "255.255.255.0"
}
```

### Apply Network Changes

```http
PUT /api2/json/nodes/{node}/network
```

### Revert Network Changes

```http
DELETE /api2/json/nodes/{node}/network
```

---

## Cluster

### Get Cluster Status

```http
GET /api2/json/cluster/status
```

**Response:**

```json
{
  "data": [
    {
      "type": "cluster",
      "name": "production",
      "nodes": 3,
      "quorate": 1,
      "version": 5
    },
    {
      "type": "node",
      "name": "node1",
      "id": "1",
      "online": 1,
      "ip": "10.0.0.1",
      "level": ""
    },
    {
      "type": "node",
      "name": "node2",
      "id": "2",
      "online": 1,
      "ip": "10.0.0.2",
      "level": ""
    }
  ]
}
```

### Get Cluster Resources

```http
GET /api2/json/cluster/resources

# Filter by type
GET /api2/json/cluster/resources?type=vm
GET /api2/json/cluster/resources?type=storage
GET /api2/json/cluster/resources?type=node
```

**Response:**

```json
{
  "data": [
    {
      "id": "qemu/100",
      "type": "qemu",
      "node": "node1",
      "vmid": 100,
      "name": "ubuntu-server",
      "status": "running",
      "mem": 4294967296,
      "maxmem": 8589934592,
      "cpu": 0.05,
      "maxcpu": 4,
      "uptime": 86400
    }
  ]
}
```

### Get Next VMID

```http
GET /api2/json/cluster/nextid
```

**Response:**

```json
{
  "data": 103
}
```

---

## Tasks

### List Tasks

```http
GET /api2/json/nodes/{node}/tasks

# With filters
GET /api2/json/nodes/{node}/tasks?vmid=100&typefilter=qmstart
```

**Response:**

```json
{
  "data": [
    {
      "upid": "UPID:node1:00001234:12345678:65432100:qmstart:100:root@pam:",
      "node": "node1",
      "pid": 1234,
      "pstart": 12345678,
      "starttime": 1699900000,
      "type": "qmstart",
      "id": "100",
      "user": "root@pam",
      "status": "OK"
    }
  ]
}
```

### Get Task Status

```http
GET /api2/json/nodes/{node}/tasks/{upid}/status
```

**Response:**

```json
{
  "data": {
    "upid": "UPID:node1:00001234:12345678:65432100:qmstart:100:root@pam:",
    "node": "node1",
    "type": "qmstart",
    "status": "running",
    "pid": 1234,
    "starttime": 1699900000
  }
}
```

### Get Task Log

```http
GET /api2/json/nodes/{node}/tasks/{upid}/log

# With pagination
GET /api2/json/nodes/{node}/tasks/{upid}/log?start=0&limit=50
```

**Response:**

```json
{
  "data": [
    {"n": 1, "t": "starting VM 100"},
    {"n": 2, "t": "started VM 100"},
    {"n": 3, "t": "TASK OK"}
  ],
  "total": 3
}
```

### Stop Task

```http
DELETE /api2/json/nodes/{node}/tasks/{upid}
```

---

## Native API Extensions

### vCluster Management

#### List vClusters

```http
GET /v1/vclusters
```

**Response:**

```json
{
  "items": [
    {
      "id": "vc-001",
      "name": "dev-cluster",
      "status": "running",
      "k8s_version": "1.28.0",
      "node": "node1",
      "isolation": "namespace",
      "created_at": "2026-01-15T10:00:00Z",
      "endpoints": {
        "api_server": "https://10.0.0.1:6443"
      }
    }
  ]
}
```

#### Create vCluster

```http
POST /v1/vclusters
Content-Type: application/json

{
  "name": "dev-cluster",
  "k8s_version": "1.28.0",
  "isolation": "namespace",
  "resources": {
    "cpu": 4,
    "memory": "8Gi"
  },
  "node_selector": {
    "node": "node1"
  }
}
```

#### Get vCluster Kubeconfig

```http
GET /v1/vclusters/{id}/kubeconfig
```

**Response:**

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://10.0.0.1:6443
    certificate-authority-data: BASE64_CA
  name: dev-cluster
contexts:
- context:
    cluster: dev-cluster
    user: admin
  name: dev-cluster
current-context: dev-cluster
users:
- name: admin
  user:
    client-certificate-data: BASE64_CERT
    client-key-data: BASE64_KEY
```

#### Scale vCluster

```http
POST /v1/vclusters/{id}/scale
Content-Type: application/json

{
  "resources": {
    "cpu": 8,
    "memory": "16Gi"
  }
}
```

#### Delete vCluster

```http
DELETE /v1/vclusters/{id}
```

### Template Management

#### List Templates

```http
GET /v1/templates
```

#### Create Template from VM

```http
POST /v1/templates
Content-Type: application/json

{
  "source_vmid": 100,
  "name": "ubuntu-22.04-base",
  "description": "Base Ubuntu 22.04 template",
  "tags": ["ubuntu", "base"]
}
```

### Metrics API

#### Get Node Metrics

```http
GET /v1/nodes/{node}/metrics

# Prometheus format
GET /v1/nodes/{node}/metrics?format=prometheus
```

**Response:**

```json
{
  "timestamp": "2026-01-15T10:00:00Z",
  "cpu": {
    "usage_percent": 15.5,
    "cores": 16,
    "load_avg": [0.5, 0.75, 0.6]
  },
  "memory": {
    "total_bytes": 68719476736,
    "used_bytes": 17179869184,
    "cached_bytes": 8589934592
  },
  "disk": {
    "read_bytes_per_sec": 1048576,
    "write_bytes_per_sec": 2097152,
    "iops_read": 100,
    "iops_write": 200
  },
  "network": {
    "rx_bytes_per_sec": 10485760,
    "tx_bytes_per_sec": 5242880,
    "rx_packets_per_sec": 10000,
    "tx_packets_per_sec": 5000
  }
}
```

### Events Stream

#### WebSocket Events

```
GET /v1/events/ws
Upgrade: websocket
```

**Event Messages:**

```json
{
  "type": "vm.status.changed",
  "timestamp": "2026-01-15T10:00:00Z",
  "data": {
    "vmid": 100,
    "node": "node1",
    "old_status": "stopped",
    "new_status": "running"
  }
}
```

---

## gRPC API

### Service Definitions

```protobuf
syntax = "proto3";

package gpve.v1;

service GuestService {
  rpc Create(CreateGuestRequest) returns (CreateGuestResponse);
  rpc Get(GetGuestRequest) returns (Guest);
  rpc List(ListGuestsRequest) returns (ListGuestsResponse);
  rpc Update(UpdateGuestRequest) returns (Guest);
  rpc Delete(DeleteGuestRequest) returns (DeleteGuestResponse);
  
  rpc Start(StartGuestRequest) returns (Operation);
  rpc Stop(StopGuestRequest) returns (Operation);
  rpc Restart(RestartGuestRequest) returns (Operation);
  
  rpc Migrate(MigrateGuestRequest) returns (Operation);
  rpc Clone(CloneGuestRequest) returns (Operation);
  
  rpc WatchStatus(WatchStatusRequest) returns (stream GuestStatus);
}

service VClusterService {
  rpc Create(CreateVClusterRequest) returns (CreateVClusterResponse);
  rpc Get(GetVClusterRequest) returns (VCluster);
  rpc List(ListVClustersRequest) returns (ListVClustersResponse);
  rpc Delete(DeleteVClusterRequest) returns (DeleteVClusterResponse);
  
  rpc GetKubeconfig(GetKubeconfigRequest) returns (GetKubeconfigResponse);
  rpc Scale(ScaleVClusterRequest) returns (Operation);
}

service ClusterService {
  rpc Status(StatusRequest) returns (ClusterStatus);
  rpc Join(JoinRequest) returns (JoinResponse);
  rpc Leave(LeaveRequest) returns (LeaveResponse);
  rpc ListNodes(ListNodesRequest) returns (ListNodesResponse);
}
```

### Example Client Usage (Go)

```go
package main

import (
    "context"
    "log"
    
    pb "github.com/go-proxmox/gpve/api/grpc/v1"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
)

func main() {
    creds, _ := credentials.NewClientTLSFromFile("ca.crt", "")
    conn, err := grpc.Dial("node1:8007", grpc.WithTransportCredentials(creds))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
    
    client := pb.NewGuestServiceClient(conn)
    
    // Create VM
    resp, err := client.Create(context.Background(), &pb.CreateGuestRequest{
        Name:   "test-vm",
        Cores:  4,
        Memory: 8192,
        Disks: []*pb.DiskConfig{
            {
                Storage: "local-lvm",
                Size:    "32G",
            },
        },
    })
    if err != nil {
        log.Fatal(err)
    }
    
    log.Printf("Created VM: %d", resp.Vmid)
    
    // Watch VM status
    stream, err := client.WatchStatus(context.Background(), &pb.WatchStatusRequest{
        Vmid: resp.Vmid,
    })
    if err != nil {
        log.Fatal(err)
    }
    
    for {
        status, err := stream.Recv()
        if err != nil {
            break
        }
        log.Printf("VM %d status: %s", resp.Vmid, status.State)
    }
}
```

---

## Error Responses

### Error Format

All API errors follow a consistent format:

```json
{
  "errors": {
    "param_name": "error description"
  },
  "message": "Human readable error message",
  "status": 400
}
```

### Common Error Codes

| HTTP Code | Description         | Example                                   |
| --------- | ------------------- | ----------------------------------------- |
| 400       | Bad Request         | Invalid parameter value                   |
| 401       | Unauthorized        | Missing or invalid authentication         |
| 403       | Forbidden           | Insufficient permissions                  |
| 404       | Not Found           | Resource does not exist                   |
| 409       | Conflict            | Resource already exists or state conflict |
| 423       | Locked              | Resource is locked by another operation   |
| 500       | Internal Error      | Server-side error                         |
| 502       | Bad Gateway         | Backend service unavailable               |
| 503       | Service Unavailable | Cluster not ready                         |

### Error Examples

#### Authentication Error

```json
{
  "message": "authentication failure",
  "status": 401
}
```

#### Permission Denied

```json
{
  "message": "Permission denied - missing 'VM.Allocate' on '/vms/100'",
  "status": 403
}
```

#### Resource Not Found

```json
{
  "message": "Configuration file 'nodes/node1/qemu-server/999.conf' does not exist",
  "status": 404
}
```

#### Validation Error

```json
{
  "errors": {
    "memory": "value must be at least 128",
    "cores": "value must be between 1 and 512"
  },
  "message": "Parameter verification failed",
  "status": 400
}
```

#### Resource Locked

```json
{
  "message": "VM 100 is locked (backup)",
  "status": 423
}
```

---

## Rate Limiting

### Headers

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1699900000
```

### Rate Limit Exceeded Response

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/json

{
  "message": "Rate limit exceeded. Try again in 60 seconds.",
  "status": 429
}
```

### Rate Limits by Endpoint Type

| Endpoint Type    | Limit | Window   |
| ---------------- | ----- | -------- |
| Authentication   | 10    | 1 minute |
| Read operations  | 1000  | 1 minute |
| Write operations | 100   | 1 minute |
| Bulk operations  | 10    | 1 minute |

---

## Pagination

### Request Parameters

```http
GET /api2/json/nodes/node1/tasks?start=0&limit=50
```

| Parameter | Default | Max | Description             |
| --------- | ------- | --- | ----------------------- |
| start     | 0       | -   | Offset from beginning   |
| limit     | 50      | 500 | Maximum items to return |

### Response Headers

```http
X-Total-Count: 1234
Link: </api2/json/nodes/node1/tasks?start=50&limit=50>; rel="next",
      </api2/json/nodes/node1/tasks?start=1200&limit=50>; rel="last"
```

---

## Filtering & Sorting

### Query Parameters

```http
GET /api2/json/cluster/resources?type=vm&status=running&sort=name&order=asc
```

| Parameter | Description          | Example                 |
| --------- | -------------------- | ----------------------- |
| type      | Resource type filter | `vm`, `storage`, `node` |
| status    | Status filter        | `running`, `stopped`    |
| node      | Node filter          | `node1`                 |
| pool      | Pool filter          | `production`            |
| sort      | Sort field           | `name`, `vmid`, `mem`   |
| order     | Sort order           | `asc`, `desc`           |

---

## Bulk Operations

### Batch VM Operations

```http
POST /v1/batch/vms/action
Content-Type: application/json

{
  "action": "start",
  "vmids": [100, 101, 102, 103],
  "options": {
    "parallel": 4,
    "continue_on_error": true
  }
}
```

**Response:**

```json
{
  "results": [
    {"vmid": 100, "success": true, "upid": "UPID:..."},
    {"vmid": 101, "success": true, "upid": "UPID:..."},
    {"vmid": 102, "success": false, "error": "VM is locked"},
    {"vmid": 103, "success": true, "upid": "UPID:..."}
  ],
  "summary": {
    "total": 4,
    "succeeded": 3,
    "failed": 1
  }
}
```

---

## Webhooks

### Configure Webhook

```http
POST /v1/webhooks
Content-Type: application/json

{
  "name": "slack-notifications",
  "url": "https://hooks.slack.com/services/xxx",
  "events": ["vm.started", "vm.stopped", "vm.migrated", "node.offline"],
  "secret": "webhook-secret-key",
  "enabled": true
}
```

### Webhook Payload

```json
{
  "id": "evt-abc123",
  "timestamp": "2026-01-15T10:00:00Z",
  "type": "vm.started",
  "data": {
    "vmid": 100,
    "name": "ubuntu-server",
    "node": "node1"
  }
}
```

### Webhook Signature Verification

```http
X-Webhook-Signature: sha256=abc123...
X-Webhook-Timestamp: 1699900000
```

Verification (pseudo-code):

```python
expected = hmac_sha256(secret, f"{timestamp}.{body}")
if not constant_time_compare(signature, expected):
    reject()
if abs(current_time - timestamp) > 300:
    reject()  # Replay attack
```

---

## SDK Examples

### Python

```python
from proxmoxer import ProxmoxAPI

# Connect
proxmox = ProxmoxAPI(
    'node1.example.com',
    user='root@pam',
    password='secret',
    verify_ssl=False
)

# List VMs
for vm in proxmox.cluster.resources.get(type='vm'):
    print(f"{vm['vmid']}: {vm['name']} ({vm['status']})")

# Create VM
proxmox.nodes('node1').qemu.create(
    vmid=101,
    name='new-vm',
    memory=4096,
    cores=2,
    scsi0='local-lvm:32',
    net0='virtio,bridge=vmbr0'
)

# Start VM
proxmox.nodes('node1').qemu(101).status.start.post()
```

### JavaScript/TypeScript

```typescript
import { Proxmox } from 'proxmox-api';

const proxmox = new Proxmox({
  host: 'node1.example.com',
  username: 'root@pam',
  password: 'secret'
});

// List VMs
const vms = await proxmox.cluster.resources.$get({ type: 'vm' });
vms.forEach(vm => {
  console.log(`${vm.vmid}: ${vm.name} (${vm.status})`);
});

// Create VM
await proxmox.nodes.$node('node1').qemu.$post({
  vmid: 101,
  name: 'new-vm',
  memory: 4096,
  cores: 2
});

// Start VM
await proxmox.nodes.$node('node1').qemu.$vmid(101).status.start.$post();
```

### Terraform

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "go-proxmox/proxmox"
      version = "~> 1.0"
    }
  }
}

provider "proxmox" {
  endpoint = "https://node1.example.com:8006"
  username = "root@pam"
  password = var.proxmox_password
}

resource "proxmox_vm" "example" {
  node_name = "node1"
  name      = "terraform-vm"
  
  cpu {
    cores   = 4
    sockets = 1
    type    = "host"
  }
  
  memory {
    dedicated = 8192
    floating  = 4096
  }
  
  disk {
    storage   = "local-lvm"
    size      = 32
    interface = "scsi0"
  }
  
  network_device {
    bridge = "vmbr0"
    model  = "virtio"
  }
  
  operating_system {
    type = "l26"
  }
  
  cloud_init {
    user_data_file_id = proxmox_file.cloud_init.id
  }
}

resource "proxmox_file" "cloud_init" {
  node_name    = "node1"
  datastore_id = "local"
  content_type = "snippets"
  
  source_raw {
    data      = file("${path.module}/cloud-init.yaml")
    file_name = "vm-cloud-init.yaml"
  }
}
```

---

## OpenAPI Specification

The complete OpenAPI 3.0 specification is available at:

```
GET /api2/json/openapi.json
```

You can also access the Swagger UI at:

```
https://<host>:8006/api-explorer/
```