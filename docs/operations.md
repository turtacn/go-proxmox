# Operations Guide

## Installation

### System Requirements

#### Minimum Requirements
| Component | Specification |
|-----------|--------------|
| CPU | 64-bit processor with VT-x/AMD-V |
| RAM | 4 GB |
| Disk | 32 GB |
| Network | 1 GbE |
| OS | Debian 12, Ubuntu 22.04, Rocky 9 |

#### Recommended for Production
| Component | Specification |
|-----------|--------------|
| CPU | Multi-core with VT-x/AMD-V and VT-d/AMD-Vi |
| RAM | 64 GB+ |
| Disk | SSD/NVMe for OS, additional storage for VMs |
| Network | 10 GbE+ with bonding |
| OS | Debian 12 (recommended) |

### Quick Install

```bash
# Download installer
curl -fsSL https://get.go-proxmox.dev | bash

# Or with specific version
curl -fsSL https://get.go-proxmox.dev | VERSION=1.0.0 bash
````

### Manual Installation

#### Debian/Ubuntu

```bash
# Add repository
curl -fsSL https://repo.go-proxmox.dev/gpg | gpg --dearmor -o /usr/share/keyrings/gpve.gpg
echo "deb [signed-by=/usr/share/keyrings/gpve.gpg] https://repo.go-proxmox.dev/apt stable main" > /etc/apt/sources.list.d/gpve.list

# Install
apt update
apt install gpve

# Initialize
gpve-init
```

#### RHEL/Rocky

```bash
# Add repository
cat > /etc/yum.repos.d/gpve.repo <<EOF
[gpve]
name=go-proxmox Repository
baseurl=https://repo.go-proxmox.dev/rpm/stable
enabled=1
gpgcheck=1
gpgkey=https://repo.go-proxmox.dev/gpg
EOF

# Install
dnf install gpve

# Initialize
gpve-init
```

#### Container Deployment

```bash
# Docker Compose (development/testing)
curl -O https://raw.githubusercontent.com/go-proxmox/gpve/main/deploy/docker/docker-compose.yml
docker compose up -d

# Kubernetes (production)
kubectl apply -f https://raw.githubusercontent.com/go-proxmox/gpve/main/deploy/kubernetes/gpve.yaml
```

### Post-Installation

```bash
# Verify installation
gpvectl version
gpvectl status

# Access Web UI
echo "https://$(hostname):8006"

# Default credentials
# Username: root@pam
# Password: (your system root password)
```

---

## Cluster Management

### Create Cluster

On the first node:

```bash
# Initialize cluster
gpvectl cluster create production

# Verify cluster
gpvectl cluster status
```

### Join Cluster

On additional nodes:

```bash
# Get join token from existing node
gpvectl cluster join-token

# Join from new node
gpvectl cluster join --token <token> --master <master-ip>:8006

# Verify
gpvectl cluster status
```

### Leave Cluster

```bash
# Migrate workloads first
gpvectl node migrate <node-name> --target <other-node>

# Leave cluster
gpvectl cluster leave --node <node-name>
```

### Cluster Status

```bash
# Overview
gpvectl cluster status

# Detailed
gpvectl cluster status --verbose

# JSON output
gpvectl cluster status -o json
```

Example output:

```
Cluster: production
Status: healthy
Quorum: 3/3 nodes online

NODES:
NAME     STATUS   ROLE     CPU    MEMORY   VMS   UPTIME
node1    online   leader   15%    45%      12    30d
node2    online   follower 22%    52%      15    30d
node3    online   follower 18%    48%      10    28d

RESOURCES:
Total CPUs: 192 cores
Total Memory: 768 GB
Total VMs: 37
```

---

## Node Management

### Node Configuration

```yaml
# /etc/gpve/node.yaml
node:
  name: node1
  
network:
  management_interface: eth0
  
storage:
  local:
    path: /var/lib/gpve
  
resources:
  # Reserve resources for system
  reserved_memory: 4096  # MB
  reserved_cpu: 2        # cores
```

### Node Operations

```bash
# View node info
gpvectl node info <node-name>

# Set node description
gpvectl node set <node-name> --description "Primary compute node"

# Enable/disable node for scheduling
gpvectl node maintenance enable <node-name>
gpvectl node maintenance disable <node-name>

# Evacuate node (migrate all VMs)
gpvectl node evacuate <node-name> --target <other-node>
```

### Node Monitoring

```bash
# Real-time stats
gpvectl node top <node-name>

# Resource usage
gpvectl node resources <node-name>

# Task history
gpvectl node tasks <node-name> --limit 50
```

---

## Storage Management

### Storage Configuration

```yaml
# /etc/gpve/storage.yaml
storage:
  - name: local
    type: dir
    path: /var/lib/gpve/images
    content:
      - images
      - iso
      - snippets
    shared: false
    
  - name: local-lvm
    type: lvmthin
    vg_name: pve
    thinpool: data
    content:
      - rootdir
      - images
    shared: false
    
  - name: ceph-pool
    type: rbd
    pool: vm-pool
    mon_host: 10.0.0.10,10.0.0.11,10.0.0.12
    username: admin
    keyring: /etc/ceph/ceph.client.admin.keyring
    content:
      - images
    shared: true
    
  - name: nfs-backup
    type: nfs
    server: 10.0.0.20
    export: /backup
    content:
      - backup
      - iso
    shared: true
```

### Storage Operations

```bash
# List storage
gpvectl storage list

# Add storage
gpvectl storage add local-zfs \
  --type zfspool \
  --pool tank/vms \
  --content images,rootdir

# Remove storage
gpvectl storage remove <storage-name>

# Check storage status
gpvectl storage status <storage-name>

# Scan storage
gpvectl storage scan <storage-name>
```

### Volume Management

```bash
# List volumes
gpvectl volume list --storage local-lvm

# Create volume
gpvectl volume create --storage local-lvm --size 100G --name data-disk

# Delete volume
gpvectl volume delete local-lvm:vm-100-disk-0

# Move volume
gpvectl volume move vm-100-disk-0 --from local-lvm --to ceph-pool
```

### Backup & Restore

```bash
# Create backup
gpvectl backup create 100 \
  --storage nfs-backup \
  --mode snapshot \
  --compress zstd

# List backups
gpvectl backup list --vmid 100

# Restore backup
gpvectl backup restore vzdump-qemu-100-2026_01_15-10_00_00.vma \
  --storage local-lvm \
  --vmid 101

# Schedule backups
gpvectl backup schedule create daily-backup \
  --schedule "0 2 * * *" \
  --storage nfs-backup \
  --pool production \
  --retention 7
```

---

## Network Management

### Bridge Configuration

```yaml
# /etc/gpve/network.yaml
network:
  bridges:
    - name: vmbr0
      comment: "Management Network"
      ports:
        - eth0
      address: 10.0.0.1/24
      gateway: 10.0.0.254
      
    - name: vmbr1
      comment: "VM Network"
      ports:
        - eth1
      vlan_aware: true
      
    - name: vmbr2
      comment: "Storage Network"
      ports:
        - eth2
        - eth3
      bond_mode: 802.3ad
      bond_xmit_hash_policy: layer3+4
      address: 10.1.0.1/24
      mtu: 9000
```

### Network Operations

```bash
# List networks
gpvectl network list

# Create bridge
gpvectl network create vmbr10 \
  --type bridge \
  --ports eth4 \
  --address 192.168.10.1/24

# Apply changes
gpvectl network apply

# Revert changes
gpvectl network revert
```

### SDN (Software-Defined Networking)

```bash
# Create VNet
gpvectl sdn vnet create prod-net \
  --zone production \
  --vlan 100

# Create subnet
gpvectl sdn subnet create prod-subnet \
  --vnet prod-net \
  --cidr 10.100.0.0/24 \
  --gateway 10.100.0.1 \
  --dhcp

# Apply SDN configuration
gpvectl sdn apply
```

### Firewall

```bash
# Enable firewall
gpvectl firewall enable

# Add cluster-wide rule
gpvectl firewall rule add \
  --direction in \
  --action accept \
  --source 10.0.0.0/8 \
  --dest-port 22 \
  --proto tcp

# Add VM-specific rule
gpvectl firewall rule add --vmid 100 \
  --direction in \
  --action accept \
  --dest-port 80,443 \
  --proto tcp

# List rules
gpvectl firewall rules list
gpvectl firewall rules list --vmid 100
```

---

## VM Management

### Create VM

```bash
# Interactive creation
gpvectl vm create

# From command line
gpvectl vm create \
  --name ubuntu-server \
  --cores 4 \
  --memory 8192 \
  --disk local-lvm:32 \
  --net virtio,bridge=vmbr0 \
  --cdrom local:iso/ubuntu-22.04.iso \
  --start

# From template
gpvectl vm clone 9000 \
  --name new-server \
  --full \
  --target node2
```

### VM Configuration

```bash
# View config
gpvectl vm config 100

# Modify config
gpvectl vm set 100 \
  --memory 16384 \
  --cores 8 \
  --cpu host

# Add disk
gpvectl vm disk add 100 \
  --storage local-lvm \
  --size 100G

# Add network
gpvectl vm net add 100 \
  --bridge vmbr1 \
  --model virtio \
  --tag 100

# Resize disk
gpvectl vm disk resize 100 --disk scsi0 --size +50G
```

### VM Operations

```bash
# Power operations
gpvectl vm start 100
gpvectl vm stop 100
gpvectl vm shutdown 100
gpvectl vm reboot 100
gpvectl vm reset 100

# With options
gpvectl vm stop 100 --timeout 120 --force
gpvectl vm shutdown 100 --force-stop

# Batch operations
gpvectl vm start 100,101,102,103
gpvectl vm stop --pool production
```

### Snapshots

```bash
# Create snapshot
gpvectl vm snapshot create 100 before-upgrade \
  --description "Before system upgrade" \
  --include-ram

# List snapshots
gpvectl vm snapshot list 100

# Rollback
gpvectl vm snapshot rollback 100 before-upgrade

# Delete
gpvectl vm snapshot delete 100 before-upgrade
```

### Migration

```bash
# Offline migration
gpvectl vm migrate 100 --target node2

# Live migration
gpvectl vm migrate 100 --target node2 --online

# With local disk
gpvectl vm migrate 100 --target node2 --online \
  --with-local-disks \
  --target-storage ceph-pool

# Bulk migration
gpvectl vm migrate --node node1 --target node2
```

### Console Access

```bash
# Serial console
gpvectl vm console 100

# VNC (opens viewer)
gpvectl vm vnc 100

# SPICE (opens viewer)
gpvectl vm spice 100

# Get VNC URL
gpvectl vm vnc 100 --url-only
```

---

## vCluster Management

### Create vCluster

```bash
# Create vCluster
gpvectl vcluster create dev-cluster \
  --k8s-version 1.28.0 \
  --cpu 4 \
  --memory 8Gi \
  --node node1

# Create with custom options
gpvectl vcluster create prod-cluster \
  --k8s-version 1.28.0 \
  --cpu 16 \
  --memory 32Gi \
  --isolation namespace \
  --high-availability
```

### vCluster Operations

```bash
# List vClusters
gpvectl vcluster list

# Get kubeconfig
gpvectl vcluster kubeconfig dev-cluster > kubeconfig.yaml
export KUBECONFIG=kubeconfig.yaml
kubectl get nodes

# Scale resources
gpvectl vcluster scale dev-cluster --cpu 8 --memory 16Gi

# Connect (port-forward)
gpvectl vcluster connect dev-cluster

# Delete
gpvectl vcluster delete dev-cluster
```

---

## High Availability

### HA Configuration

```yaml
# /etc/gpve/ha.yaml
ha:
  enabled: true
  
  fencing:
    enabled: true
    devices:
      - type: ipmi
        address: 10.0.100.1
        username: admin
        password_file: /etc/gpve/ipmi-password
        
      - type: watchdog
        device: /dev/watchdog
        
  policy:
    default: migrate
    shutdown_policy: conditional
    
  groups:
    - name: prefer-node1
      nodes:
        - node: node1
          priority: 2
        - node: node2
          priority: 1
      nofailback: false
      restricted: false
```

### HA Management

```bash
# Enable HA for VM
gpvectl ha add 100 --group prefer-node1 --max-restart 3 --max-relocate 2

# List HA resources
gpvectl ha list

# Check HA status
gpvectl ha status

# Remove from HA
gpvectl ha remove 100

# Test fencing
gpvectl ha fence test node1
```

### Fencing Configuration

```bash
# Add fencing device
gpvectl ha fence add node1 \
  --type ipmi \
  --address 10.0.100.1 \
  --username admin \
  --password-file /etc/gpve/ipmi-password

# Test fence device
gpvectl ha fence test node1 --device ipmi

# List fence devices
gpvectl ha fence list
```

---

## Monitoring & Alerting

### Built-in Monitoring

```bash
# Real-time cluster stats
gpvectl top

# Node metrics
gpvectl node top node1

# VM metrics
gpvectl vm top 100

# Storage metrics
gpvectl storage stats
```

### Prometheus Integration

```yaml
# /etc/gpve/prometheus.yaml
metrics:
  enabled: true
  listen: ":9090"
  path: "/metrics"
  
  labels:
    cluster: production
    environment: prod
```

Prometheus scrape config:

```yaml
scrape_configs:
  - job_name: 'gpve'
    static_configs:
      - targets:
        - 'node1:9090'
        - 'node2:9090'
        - 'node3:9090'
    metrics_path: '/metrics'
    scheme: 'https'
    tls_config:
      ca_file: '/etc/prometheus/gpve-ca.crt'
```

### Grafana Dashboards

```bash
# Import official dashboards
gpvectl monitoring dashboards import --grafana http://grafana:3000

# Dashboard IDs:
# - GPVE Cluster Overview: 20001
# - GPVE Node Details: 20002
# - GPVE VM Details: 20003
# - GPVE Storage: 20004
```

### Alerting

```yaml
# /etc/gpve/alerts.yaml
alerts:
  enabled: true
  
  handlers:
    - name: email
      type: smtp
      config:
        server: smtp.example.com
        port: 587
        from: alerts@example.com
        to:
          - ops@example.com
          
    - name: slack
      type: webhook
      config:
        url: https://hooks.slack.com/services/xxx
        
    - name: pagerduty
      type: pagerduty
      config:
        service_key: xxx
        
  rules:
    - name: node_down
      condition: "node_up == 0"
      duration: 1m
      severity: critical
      handlers:
        - pagerduty
        - slack
        
    - name: high_cpu
      condition: "node_cpu_usage > 90"
      duration: 5m
      severity: warning
      handlers:
        - slack
        
    - name: disk_space_low
      condition: "storage_available_percent < 10"
      duration: 1m
      severity: warning
      handlers:
        - email
```

---

## Security

### TLS Configuration

```bash
# Generate new certificates
gpvectl cert renew

# View certificate info
gpvectl cert info

# Import custom CA
gpvectl cert import-ca /path/to/ca.crt

# Generate client certificate
gpvectl cert generate-client --name automation --days 365
```

### User Management

```bash
# Create user
gpvectl user create john@pam \
  --password \
  --email john@example.com \
  --groups admins

# Create API token
gpvectl user token create john@pam automation \
  --expire 365d \
  --privsep

# List users
gpvectl user list

# Modify user
gpvectl user set john@pam --groups "admins,developers"

# Delete user
gpvectl user delete john@pam
```

### RBAC

```bash
# Create role
gpvectl role create vm-operator \
  --privs "VM.Audit,VM.PowerMgmt,VM.Console"

# Assign role
gpvectl acl add / --user john@pam --role vm-operator

# Assign at specific path
gpvectl acl add /vms/100 --user john@pam --role Administrator

# List ACLs
gpvectl acl list

# Remove ACL
gpvectl acl remove /vms/100 --user john@pam
```

### Audit Logging

```yaml
# /etc/gpve/audit.yaml
audit:
  enabled: true
  log_file: /var/log/gpve/audit.log
  
  events:
    - auth.*
    - vm.create
    - vm.delete
    - vm.start
    - vm.stop
    - cluster.*
    - user.*
    - acl.*
    
  format: json
  
  # Forward to external SIEM
  forward:
    type: syslog
    address: siem.example.com:514
    protocol: tcp
    tls: true
```

```bash
# View audit log
gpvectl audit log --since 1h

# Search audit log
gpvectl audit search --user john@pam --action "vm.*"

# Export audit log
gpvectl audit export --since 2026-01-01 --format json > audit.json
```

---

## Troubleshooting

### Common Issues

#### Node Not Joining Cluster

```bash
# Check connectivity
ping <master-node>
nc -zv <master-node> 8006
nc -zv <master-node> 2379

# Check time sync
timedatectl status
chronyc sources

# Check certificates
gpvectl cert verify

# Check logs
journalctl -u gpve-server -f
```

#### VM Won't Start

```bash
# Check VM status
gpvectl vm status 100 --verbose

# Check QEMU logs
cat /var/log/gpve/qemu/100.log

# Verify storage
gpvectl storage status local-lvm

# Check resource availability
gpvectl node resources

# Try starting manually
qemu-system-x86_64 -readconfig /etc/gpve/qemu-server/100.conf
```

#### Storage Issues

```bash
# Check storage health
gpvectl storage health

# Verify LVM
lvs
vgs
pvs

# Check ZFS
zpool status
zfs list

# Check Ceph
ceph -s
ceph osd tree
```

#### Network Issues

```bash
# Check bridges
ip link show
bridge link show

# Check connectivity
ping -I vmbr0 <gateway>

# Check firewall
nft list ruleset
gpvectl firewall status

# Check VM network
gpvectl vm exec 100 -- ip addr
gpvectl vm exec 100 -- ping 8.8.8.8
```

### Diagnostic Commands

```bash
# Full system diagnostics
gpvectl diagnose

# Generate support bundle
gpvectl support-bundle --output /tmp/support.tar.gz

# Check service status
gpvectl service status

# Verify configuration
gpvectl config verify

# Check cluster health
gpvectl cluster health
```

### Log Locations

| Log      | Path                          | Purpose              |
| -------- | ----------------------------- | -------------------- |
| Server   | /var/log/gpve/gpve-server.log | API server logs      |
| Agent    | /var/log/gpve/gpve-agent.log  | Node agent logs      |
| Tasks    | /var/log/gpve/tasks/*.log     | Individual task logs |
| QEMU     | /var/log/gpve/qemu/*.log      | VM-specific logs     |
| Firewall | /var/log/gpve/firewall.log    | Firewall events      |
| Audit    | /var/log/gpve/audit.log       | Security audit       |

### Debug Mode

```bash
# Enable debug logging
gpvectl config set logging.level debug
systemctl restart gpve-server gpve-agent

# Or via environment
GPVE_DEBUG=true GPVE_LOG_LEVEL=debug gpve-server

# Component-specific debug
GPVE_DEBUG=qemu,storage,network gpve-agent
```

---

## Upgrade & Maintenance

### Rolling Upgrade

```bash
# Check current version
gpvectl version

# Check available updates
gpvectl upgrade check

# Upgrade cluster (rolling)
gpvectl upgrade start --version 1.2.0

# Monitor upgrade progress
gpvectl upgrade status

# Upgrade single node
gpvectl upgrade node node1 --version 1.2.0
```

### Manual Upgrade

```bash
# On each node (start with non-leader nodes):

# 1. Enable maintenance mode
gpvectl node maintenance enable $(hostname)

# 2. Wait for migrations to complete
gpvectl node wait-empty $(hostname)

# 3. Upgrade packages
apt update && apt upgrade gpve

# 4. Restart services
systemctl restart gpve-server gpve-agent

# 5. Verify
gpvectl version
gpvectl cluster status

# 6. Disable maintenance mode
gpvectl node maintenance disable $(hostname)
```

### Database Maintenance

```bash
# Compact etcd
gpvectl etcd compact

# Defragment etcd
gpvectl etcd defrag

# Backup etcd
gpvectl etcd backup /backup/etcd-$(date +%Y%m%d).db

# Restore etcd (emergency)
gpvectl etcd restore /backup/etcd-20260115.db
```

### Health Checks

```bash
# Full health check
gpvectl health check

# Component checks
gpvectl health check --component api
gpvectl health check --component storage
gpvectl health check --component network
gpvectl health check --component cluster
```

---

## Backup & Disaster Recovery

### Cluster Backup

```bash
# Backup cluster configuration
gpvectl backup cluster --output /backup/cluster-$(date +%Y%m%d).tar.gz

# Includes:
# - etcd data
# - Node configurations
# - User/ACL database
# - Storage definitions
# - Network configurations
```

### Disaster Recovery

```bash
# 1. Install fresh system
gpve-init --skip-cluster

# 2. Restore cluster configuration
gpvectl restore cluster /backup/cluster-20260115.tar.gz

# 3. Restore VMs from backups
gpvectl backup restore-all --storage backup-storage

# 4. Verify
gpvectl cluster status
gpvectl vm list
```

### Geo-Replication

```yaml
# /etc/gpve/replication.yaml
replication:
  enabled: true
  
  targets:
    - name: dr-site
      endpoint: https://dr-node1:8006
      token: <api-token>
      
  jobs:
    - name: replicate-production
      source_pool: production
      target: dr-site
      schedule: "*/15 * * * *"  # Every 15 minutes
      rate_limit: 100M
```

```bash
# Manual replication
gpvectl replication sync --pool production --target dr-site

# Check replication status
gpvectl replication status

# Failover to DR site
gpvectl replication failover --target dr-site
```


---