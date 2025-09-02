# Proxmox Backup Strategy, 3-2-1 Rule & Resource Management

## Proxmox Backup Features

### Backup Types

#### 1. **Snapshot Mode** (Default for VMs with qemu-agent)
- Creates point-in-time snapshot while VM is running
- Minimal downtime (milliseconds)
- Requires guest agent for filesystem consistency
- Best for: Production VMs that cannot tolerate downtime

#### 2. **Suspend Mode**
- Suspends VM, saves RAM state, performs backup
- Short downtime (seconds to minutes)
- Guarantees consistency without guest agent
- Best for: Critical databases without agent support

#### 3. **Stop Mode**
- Stops VM completely during backup
- Longest downtime but highest consistency
- No special requirements
- Best for: Maintenance windows, critical data

### Backup Formats

#### VMA (Virtual Machine Archive)
- Proxmox native format
- Contains VM config + disk data
- Supports compression (LZO, GZIP, ZSTD)

#### PBS (Proxmox Backup Server) Format
- Incremental forever strategy
- Deduplication at chunk level (4MB chunks)
- Encryption support
- Typical dedup ratios: 3:1 to 10:1

### Compression Options
```
None     - Fastest, largest size
LZO      - Fast compression, moderate size (default)
GZIP     - Slower, better compression
ZSTD     - Best balance of speed and compression
```

## Implementing 3-2-1 Backup Rule

### The 3-2-1 Rule Explained
- **3** copies of data (1 primary + 2 backups)
- **2** different storage media types
- **1** offsite copy

### Proxmox 3-2-1 Implementation

```
┌──────────────────────────────────────────────────────┐
│                  3-2-1 Architecture                   │
├──────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────┐                                     │
│  │ Primary PVE │  Copy 1: Production VMs             │
│  │   Server    │  Storage: ZFS/Ceph/SAN              │
│  └──────┬──────┘                                     │
│         │                                             │
│         ├────────────────────────┐                   │
│         ▼                        ▼                   │
│  ┌─────────────┐          ┌─────────────┐           │
│  │  Local PBS  │          │ Remote PBS  │           │
│  │   Server    │          │   Server    │           │
│  └─────────────┘          └─────────────┘           │
│   Copy 2: Local            Copy 3: Offsite          │
│   Media: NAS/SAN           Media: Cloud/Colo        │
│                                                       │
└──────────────────────────────────────────────────────┘
```

### Configuration Example

#### 1. Local Backup (Copy 2)
```bash
# PBS Storage Configuration
pvesm add pbs local-pbs \
  --server 10.0.1.50 \
  --datastore local-backups \
  --username backup@pbs \
  --password <password> \
  --fingerprint <fingerprint>

# Backup Job
pvesh create /cluster/backup \
  --node pve1 \
  --storage local-pbs \
  --schedule "02:00" \
  --mode snapshot \
  --compress zstd \
  --mailto admin@company.com
```

#### 2. Remote/Offsite Backup (Copy 3)
```bash
# Remote PBS or S3-compatible storage
pvesm add pbs remote-pbs \
  --server remote.backup.site \
  --datastore offsite-backups \
  --username backup@pbs \
  --password <password> \
  --fingerprint <fingerprint>

# Sync job on PBS server
proxmox-backup-manager sync-job create \
  --id offsite-sync \
  --schedule "06:00" \
  --remote remote-site \
  --remote-store backups \
  --store local-backups \
  --remove-vanished true
```

#### 3. Cloud Storage Integration (Alternative Copy 3)
```bash
# Using rclone for cloud backup
# /etc/systemd/system/pbs-to-cloud.service
[Unit]
Description=Sync PBS to Cloud Storage
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/rclone sync \
  /mnt/datastore/backups \
  s3:company-proxmox-backups \
  --config /etc/rclone/rclone.conf

[Timer]
OnCalendar=daily
OnCalendar=06:00
```

## Backup Scheduling Strategy

### Production Schedule Template

```yaml
Critical VMs (Tier 1):
  Schedule: Every 4 hours
  Mode: Snapshot
  Retention: 
    - Hourly: Keep 24
    - Daily: Keep 14
    - Weekly: Keep 8
    - Monthly: Keep 12

Important VMs (Tier 2):
  Schedule: Daily at 02:00
  Mode: Snapshot
  Retention:
    - Daily: Keep 7
    - Weekly: Keep 4
    - Monthly: Keep 6

Standard VMs (Tier 3):
  Schedule: Weekly
  Mode: Snapshot
  Retention:
    - Weekly: Keep 4
    - Monthly: Keep 3
```

### Staggered Backup Windows
```
00:00-02:00  - Database servers (Tier 1)
02:00-04:00  - Application servers (Tier 2)
04:00-06:00  - Web servers (Tier 2)
06:00-07:00  - Development VMs (Tier 3)
07:00-08:00  - Verification jobs
```

### PBS Retention Policy Configuration
```bash
# Configure retention per backup job
proxmox-backup-manager datastore update <datastore> \
  --keep-last 5 \
  --keep-hourly 24 \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --keep-yearly 2

# Prune schedule
proxmox-backup-manager prune-job create \
  --id daily-prune \
  --store local-backups \
  --schedule "03:30"

# Garbage collection
proxmox-backup-manager garbage-collection-job update \
  --store local-backups \
  --schedule "sun 04:00"
```

## Resource Management

### CPU Resource Management

#### CPU Units (Shares)
```bash
# Default: 1024
# Range: 2-262144
# Higher value = more CPU time when contention exists

# High priority VM
qm set 100 --cpuunits 2048

# Low priority VM  
qm set 101 --cpuunits 512
```

#### CPU Limits
```bash
# Limit VM to 2.5 cores worth of CPU time
qm set 100 --cpulimit 2.5

# No limit (use all available)
qm set 100 --cpulimit 0
```

#### NUMA Configuration
```bash
# Enable NUMA for large VMs
qm set 100 --numa 1

# Pin to specific NUMA node
qm set 100 --numanode 0
```

### Memory Resource Management

#### Memory Allocation
```bash
# Fixed memory allocation
qm set 100 --memory 8192

# Ballooning (dynamic memory)
qm set 100 --memory 8192 --balloon 2048
# Min: 2GB, Max: 8GB
```

#### Memory Shares
```bash
# Priority for memory under pressure
# Range: 0-50000, default: 1000
qm set 100 --shares 2000
```

#### Huge Pages
```bash
# Enable huge pages for better performance
echo "vm.nr_hugepages = 1024" >> /etc/sysctl.conf
sysctl -p

# Configure VM to use huge pages
qm set 100 --hugepages 1024
```

### Storage I/O Management

#### I/O Throttling
```bash
# Limit disk I/O
# MB/s limits
qm set 100 --ide0 local-lvm:vm-100-disk-0,\
  mbps_rd=50,\
  mbps_wr=50,\
  mbps_rd_max=100,\
  mbps_wr_max=100

# IOPS limits  
qm set 100 --ide0 local-lvm:vm-100-disk-0,\
  iops_rd=1000,\
  iops_wr=1000,\
  iops_rd_max=2000,\
  iops_wr_max=2000
```

#### I/O Thread & Queue Settings
```bash
# Enable I/O thread for better performance
qm set 100 --ide0 local-lvm:vm-100-disk-0,iothread=1

# Set cache mode
# Options: none, writethrough, writeback, unsafe, directsync
qm set 100 --ide0 local-lvm:vm-100-disk-0,cache=writeback

# Async I/O mode
# Options: threads, native
qm set 100 --ide0 local-lvm:vm-100-disk-0,aio=native
```

### Network Resource Management

#### Network Rate Limiting
```bash
# Limit network bandwidth (MB/s)
qm set 100 --net0 virtio=AA:BB:CC:DD:EE:FF,\
  bridge=vmbr0,\
  rate=100  # 100 MB/s limit
```

#### Network Queues
```bash
# Multi-queue for better performance
qm set 100 --net0 virtio=AA:BB:CC:DD:EE:FF,\
  bridge=vmbr0,\
  queues=4  # 4 queues for 4 vCPUs
```

### Resource Pools

#### Create Resource Pool
```bash
# Create pool for department/project
pvesh create /pools --poolid production

# Add VMs to pool
pvesh set /pools/production --vms 100,101,102

# Add storage to pool  
pvesh set /pools/production --storage local-lvm,nfs-storage
```

#### Set Pool Permissions
```bash
# Create group
pveum group add production-admins

# Assign pool permissions
pveum acl modify /pool/production \
  --group production-admins \
  --role PVEAdmin
```

### Resource Monitoring

#### Built-in Metrics
```bash
# Enable metrics server
cat >> /etc/pve/status.cfg <<EOF
influxdb:
  server monitoring.local
  port 8086
  protocol udp
EOF

# Or Graphite
cat >> /etc/pve/status.cfg <<EOF
graphite:
  server monitoring.local
  port 2003
  protocol tcp
  path proxmox
EOF
```

#### Resource Usage Commands
```bash
# Check VM resource usage
qm status 100 --verbose

# Monitor real-time usage
qm monitor 100
# Then type: info status

# Check cluster resources
pvesh get /cluster/resources

# Check node statistics
pvesh get /nodes/pve1/status
```

### Backup Resource Optimization

#### Backup Performance Settings
```bash
# Configure backup bandwidth limit
cat >> /etc/vzdump.conf <<EOF
# Bandwidth limit in KB/s
bwlimit: 102400  # 100 MB/s

# I/O limit for backup
ionice: 7

# Concurrent backups
maxfiles: 2

# Compression threads
pigz: 4
zstd: 4
EOF
```

#### PBS Performance Tuning
```bash
# Tune PBS datastore
proxmox-backup-manager datastore update local-backups \
  --chunk-size 4096 \
  --workers 4

# Configure sync job limits
proxmox-backup-manager sync-job update offsite-sync \
  --rate-limit 50000  # 50 MB/s
```

## Backup Verification Strategy

### Automated Verification
```bash
#!/bin/bash
# /usr/local/bin/verify-backups.sh

# Verify latest backups
proxmox-backup-manager verify-job create \
  --id daily-verify \
  --store local-backups \
  --schedule "07:00" \
  --outdated-after 7 \
  --ignore-verified false

# Test restore to isolated environment
TEST_NODE="pve-test"
BACKUP_ID=$(proxmox-backup-client list --repository ... | head -1)

qmrestore $BACKUP_ID 999 \
  --storage test-storage \
  --node $TEST_NODE \
  --start 0

# Run smoke tests
ssh $TEST_NODE "qm start 999 && sleep 60 && qm status 999"

# Cleanup
ssh $TEST_NODE "qm destroy 999"
```

### Recovery Time Metrics
```yaml
Recovery Objectives:
  Tier 1 (Critical):
    RTO: < 15 minutes
    RPO: < 1 hour
    Test Frequency: Monthly
    
  Tier 2 (Important):
    RTO: < 2 hours
    RPO: < 4 hours
    Test Frequency: Quarterly
    
  Tier 3 (Standard):
    RTO: < 8 hours
    RPO: < 24 hours
    Test Frequency: Bi-annually
```

## Best Practices Summary

### Backup Best Practices
1. Test restores regularly (monthly minimum)
2. Monitor backup job completion rates
3. Verify backup integrity automatically
4. Document restore procedures
5. Keep backup encryption keys secure but accessible
6. Monitor storage capacity (alert at 80% full)
7. Use dedicated backup network if possible
8. Implement backup job dependencies

### Resource Management Best Practices
1. Overcommit CPU but not memory
2. Use CPU units for fair scheduling
3. Enable ballooning for memory flexibility
4. Implement I/O limits for shared storage
5. Use resource pools for multi-tenancy
6. Monitor resource trends for capacity planning
7. Set up alerts for resource exhaustion
8. Regular review and adjustment of limits