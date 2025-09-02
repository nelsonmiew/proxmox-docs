# Proxmox Production Setup - Task List & Pre-requirements

## Pre-requirements

### 1. Hardware Assessment and Compatibility Check
- **CPU**: 64-bit processor with Intel VT/AMD-V virtualization support
- **RAM**: Minimum 8GB, recommended 32GB+ for production
- **Storage**: Enterprise-grade SSDs or NVMe drives
- **Network**: Minimum 2x 1Gbps NICs (preferably 10Gbps for production)
- **IPMI/iDRAC**: Remote management capability
- Verify hardware on Proxmox HCL (Hardware Compatibility List)

### 2. Network Infrastructure Planning
- **IP Allocation**:
  - Management network IPs for Proxmox hosts
  - VM network ranges
  - Storage network (if separate)
  - Backup network (recommended separate)
- **VLANs**: Plan VLAN segregation for management, VMs, storage
- **DNS**: Internal DNS server for hostname resolution
- **Firewall rules**: Define required ports (8006 for web UI, 22 for SSH, etc.)

### 3. Storage Planning and RAID Configuration
- **Boot drives**: RAID1 for OS (2x SSDs minimum)
- **VM storage**: 
  - ZFS recommended for data integrity
  - RAID10 for performance or RAIDZ2 for redundancy
- **Backup storage**: Separate storage pool/array
- Calculate IOPS requirements based on workload

### 4. Licensing and Subscription Planning
- Proxmox VE Subscription (for production support and stable updates)
- Proxmox Backup Server Subscription
- Consider: Basic, Standard, or Premium based on SLA requirements

## Primary Server Setup Tasks

### 5. Install Proxmox VE on Main Server
- Download latest Proxmox VE ISO
- Create bootable USB with Rufus/Etcher
- Install with appropriate disk configuration
- Set hostname in FQDN format (e.g., pve1.domain.local)

### 6. Initial Network Configuration
- Configure management interface
- Setup bridge for VM networking (vmbr0)
- Configure additional bridges for storage/backup networks
- Enable jumbo frames if using 10Gbps
- Configure bond interfaces for redundancy

### 7. Storage Configuration (ZFS/LVM)
- Create ZFS pools for VM storage
- Configure appropriate ARC size
- Set up thin provisioning if needed
- Configure local-lvm for container storage
- Mount additional storage paths

### 8. Security Hardening
- Change root password to strong password
- Configure SSH key authentication
- Disable root SSH password login
- Setup fail2ban
- Configure firewall rules in Proxmox
- Enable and configure 2FA for web UI
- Regular security updates schedule
- Configure SSL certificates

## Backup Server Setup Tasks

### 9. Install Proxmox Backup Server
- Install on dedicated hardware or VM
- Minimum requirements: 4GB RAM, 32GB boot disk
- Configure with sufficient storage for retention policy

### 10. Configure Storage Repositories
- Create datastores for different backup types
- Configure deduplication settings
- Set up garbage collection schedules
- Configure prune schedules based on retention policy

## Integration Tasks

### 11. Connect PVE to PBS
- Add PBS as storage in Proxmox VE
- Configure authentication (API tokens recommended)
- Test connectivity
- Configure encryption if required

### 12. Configure Backup Schedules and Retention Policies
- Define backup windows
- Set up daily/weekly/monthly schedules
- Configure retention:
  - Daily: 7 backups
  - Weekly: 4 backups
  - Monthly: 12 backups
  - Yearly: 2 backups (adjust as needed)
- Configure backup modes (snapshot/suspend/stop)

## Advanced Setup (Optional)

### 13. High Availability Setup (if multiple nodes)
- Minimum 3 nodes for quorum
- Configure shared storage (Ceph, NFS, or iSCSI)
- Create cluster
- Configure fencing devices
- Set up HA groups and priorities

### 14. Monitoring Setup
- Configure built-in metrics server
- Set up external monitoring (Zabbix/Prometheus)
- Configure email alerts
- Set up SMS/webhook notifications for critical alerts
- Monitor:
  - CPU/Memory/Storage usage
  - Network throughput
  - VM performance
  - Backup job status

## Final Tasks

### 15. Documentation
- Network diagram
- IP allocation spreadsheet
- Backup/restore procedures
- Disaster recovery runbook
- Admin credentials (store securely)
- Change management procedures

### 16. Verify Backup/Restore Procedures
- Test full VM restore
- Test file-level restore
- Test bare-metal recovery
- Document restore times (RTO)
- Verify data integrity

### 17. Performance Testing and Optimization
- Run benchmark tests
- Optimize VM settings
- Tune kernel parameters if needed
- Configure NUMA if applicable
- Test failover scenarios

## Important Considerations

### Network Ports to Open
- 8006: Proxmox VE web interface
- 8007: Proxmox Backup Server web interface  
- 22: SSH
- 3128: SPICE proxy
- 5900-5999: VNC console
- 111, 2049: NFS (if used)
- 3260: iSCSI (if used)

### Maintenance Windows
- Schedule regular maintenance windows for:
  - Updates and patches
  - Hardware maintenance
  - Backup verification
  - Performance reviews

### Disaster Recovery Planning
- Off-site backup copies
- Recovery time objective (RTO)
- Recovery point objective (RPO)
- Regular DR drills
- Contact list for emergencies

### Best Practices
- Never run Proxmox VE without subscription in production
- Always test updates in non-production first
- Keep at least 20% free space on all storage
- Regular monitoring of system logs
- Document all changes
- Use VLANs to segregate traffic
- Implement defense in depth security