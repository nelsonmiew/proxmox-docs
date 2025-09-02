# Proxmox Production Deployment Guide

## Overview

This comprehensive guide provides everything needed to deploy, secure, and maintain a production-grade Proxmox Virtual Environment (VE) infrastructure. It covers enterprise-level configurations for high availability, backup strategies, security hardening, monitoring, and operational best practices.

### Target Audience

- System Administrators
- Infrastructure Engineers
- DevOps Teams
- IT Managers planning virtualization deployments

### Infrastructure Scope

- Primary Proxmox VE cluster setup
- Proxmox Backup Server (PBS) implementation
- High Availability (HA) configuration
- Enterprise VPN integration
- Comprehensive monitoring stack
- Security hardening and access control

## Quick Start

1. Review [Prerequisites & Planning](./proxmox-production-setup.md)
2. Follow [Security & Network Setup](./security-networking-access.md)
3. Implement [Backup Strategy](./backup-strategy-and-resources.md)
4. Deploy [Monitoring Stack](./monitoring-observability-stack.md)
5. Configure [VPN Access](./fortigate-vpn-integration.md)
6. Deploy [Production Applications](./application-deployment-examples.md)

## Documentation Index

### 1. [Production Setup & Prerequisites](./proxmox-production-setup.md)

**Building a rock-solid foundation is crucial for any production environment. This guide ensures you start with the right hardware, network design, and configuration choices that will support your infrastructure for years to come. Making the right decisions upfront saves countless hours of troubleshooting and prevents costly downtime later.**

**Why This Matters:**

- Proper hardware selection prevents performance bottlenecks and ensures reliability under load
- Network segmentation from day one provides security isolation and performance optimization
- Storage architecture decisions directly impact VM performance and backup speeds
- Licensing ensures you get enterprise support when critical issues arise

**Key Topics:**

- Hardware requirements and compatibility verification
- Network infrastructure planning with proper VLAN segmentation
- Storage architecture selection (ZFS for data integrity, Ceph for scale-out, NFS for simplicity)
- Licensing and subscription planning for production support
- Initial server configuration with security in mind
- Security hardening checklist to protect against common threats
- High availability setup for zero-downtime maintenance
- Disaster recovery planning before disaster strikes

**Critical Decisions:**

- Minimum 3 nodes for HA quorum (2 nodes = split-brain risk)
- Separate networks prevent backup traffic from impacting production
- Enterprise subscription provides access to stable repository and support

---

### 2. [Backup Strategy & Resource Management](./backup-strategy-and-resources.md)

**Backups are your insurance policy against data loss, ransomware, and human error. Following the proven 3-2-1 strategy ensures you have hot backups (instantly available), warm backups (quickly accessible), and cold archives (safe from corruption). This guide shows how to implement bulletproof backup protection while optimizing resource usage to prevent backup operations from impacting production performance.**

**Why This Matters:**

- Without proper backups, a single hardware failure can destroy years of data
- The 3-2-1 rule has saved countless organizations from ransomware attacks
- Automated verification ensures your backups actually work when needed
- Resource management prevents backup jobs from slowing down production VMs

**Key Topics:**

- Understanding backup modes and choosing the right one for each workload
- 3-2-1 rule implementation (3 copies, 2 different media, 1 offsite)
- PBS configuration with deduplication to save 70-90% storage space
- Incremental forever strategy eliminating full backup windows
- Smart scheduling to avoid business hours impact
- Resource limits ensuring backups don't starve production VMs
- Automated verification because untested backups equal no backups
- Recovery time objectives (RTO) and recovery point objectives (RPO) planning

**Real-World Benefits:**

- Deduplication typically achieves 3:1 to 10:1 space savings
- Incremental backups complete in minutes vs hours for full backups
- Automated testing catches backup failures before you need them
- Proper resource management maintains VM performance during backups

---

### 3. [Security, Networking & Access Control](./security-networking-access.md)

**Security isn't optional in production environments. A single misconfiguration can expose your entire infrastructure to attackers. This guide implements defense-in-depth with multiple security layers, ensuring that even if one layer fails, others protect your systems. Every production Proxmox deployment has been targeted by attackers - this guide ensures yours survives.**

**Why This Matters:**

- Exposed Proxmox interfaces are actively scanned and attacked within minutes
- Proper network segmentation contains breaches and prevents lateral movement
- VPN access provides secure remote management without internet exposure
- Certificate management prevents man-in-the-middle attacks

**Key Topics:**

- Multi-layer firewall strategy blocking attacks at perimeter and host level
- VPN configuration for secure remote access without exposure
- Reverse proxy setup hiding Proxmox behind secure gateways
- Load balancer configuration for high availability and DDoS protection
- Automated SSL/TLS certificates eliminating manual renewal
- Port security preventing unauthorized service exposure
- Access control matrices defining who can access what
- Break-glass emergency procedures for disaster scenarios

**Security Principles:**

- **NEVER expose port 8006 to the internet** - it will be attacked
- Defense-in-depth means multiple barriers between attackers and systems
- Zero-trust approach - verify everything, trust nothing
- Regular updates patch vulnerabilities before they're exploited
- Audit logs provide forensic evidence after incidents

---

### 4. [FortiGate VPN Integration](./fortigate-vpn-integration.md)

**Enterprise VPN isn't just about remote access - it's about ensuring only authorized, compliant, and secure devices can connect to your infrastructure. FortiGate provides military-grade encryption, advanced threat protection, and detailed visibility into who's accessing what. This integration transforms basic remote access into enterprise-grade secure connectivity.**

**Why This Matters:**

- Consumer VPNs lack enterprise features like device compliance checking
- FortiGate's UTM features block malware before it reaches your network
- Centralized management scales to thousands of users effortlessly
- Integration with AD/LDAP provides single sign-on convenience

**Key Topics:**

- SSL VPN portal providing clientless access to resources
- IPSec site-to-site connecting multiple datacenters securely
- Zero Trust Network Access ensuring least-privilege access
- FortiClient ensuring endpoint compliance before connection
- Multi-factor authentication preventing credential compromise
- Session recording for compliance and security audits
- High availability ensuring VPN access during failures
- FortiAnalyzer providing detailed analytics and forensics

**Enterprise Benefits:**

- Host checking ensures only patched, antivirus-protected devices connect
- Application-level control prevents unauthorized service access
- Bandwidth management ensures VPN doesn't saturate links
- Detailed logging satisfies compliance requirements
- Hardware acceleration handles thousands of concurrent users

---

### 5. [Monitoring & Observability Stack](./monitoring-observability-stack.md)

**You can't fix what you can't see. Comprehensive monitoring transforms reactive firefighting into proactive problem prevention. This stack provides complete visibility from hardware sensors to application metrics, enabling you to detect issues hours or days before they impact users. The difference between good and great infrastructure teams is their monitoring.**

**Why This Matters:**

- Most outages have warning signs hours or days in advance
- Proper monitoring reduces mean time to resolution (MTTR) by 80%
- Capacity planning based on trends prevents emergency upgrades
- Security monitoring detects breaches while they're still containable

**Key Topics:**

- Prometheus collecting thousands of metrics per second efficiently
- Grafana visualizing complex data in intuitive dashboards
- Log aggregation centralizing millions of events for analysis
- Distributed tracing showing exactly where slowdowns occur
- Intelligent alerting preventing alert fatigue while catching issues
- Multi-channel notifications ensuring critical alerts are never missed
- Custom dashboards for different audiences (technical vs management)
- Performance baselines identifying abnormal behavior
- Capacity forecasting preventing resource exhaustion

**Monitoring Benefits:**

- Detect failing disks before data loss occurs
- Identify memory leaks before they cause crashes
- Spot security breaches through anomaly detection
- Predict capacity needs months in advance
- Reduce troubleshooting time from hours to minutes
- Provide SLA compliance reporting automatically

---

### 6. [Application Deployment Examples](./application-deployment-examples.md)

**Real-world production applications need more than just VMs - they need properly configured web servers, databases, caching layers, and monitoring. This guide provides battle-tested configurations for PHP (CakePHP/Laravel), Python (Django/Flask), and containerized microservices that scale from startup to enterprise. Every example includes security hardening, performance optimization, and monitoring integration.**

**Why This Matters:**

- Copy-paste ready configurations save weeks of trial and error
- Production-tested settings prevent common pitfalls and performance issues
- Security configurations protect against OWASP Top 10 vulnerabilities
- Containerization examples show modern DevOps best practices
- Load balancing configurations ensure high availability from day one

**Application Examples:**

- **CakePHP E-Commerce**: Full LAMP stack with MySQL clustering, Redis caching, and Apache optimization
- **Django SaaS Platform**: Multi-tenant architecture with PostgreSQL, Celery workers, and Nginx
- **Docker Microservices**: Container orchestration with Docker Compose and Swarm
- **Load Balancing**: HAProxy configuration for distributing traffic across instances
- **Performance Tuning**: Kernel optimizations, caching strategies, and database tuning
- **CI/CD Integration**: Automated deployment pipelines with health checks

**Production Features:**

- Auto-scaling based on load metrics
- Zero-downtime deployments with rolling updates
- Database replication and failover
- Session persistence across instances
- Centralized logging and monitoring
- Automated backup and recovery procedures

---

## Architecture Overview

```
Internet
    │
    ├─── FortiGate Firewall (HA Pair)
    │           │
    │           ├─── SSL VPN Portal
    │           ├─── IPSec Tunnels
    │           └─── ZTNA Gateway
    │
    ├─── DMZ Network (VLAN 10)
    │       ├─── Reverse Proxy (Nginx/Traefik)
    │       └─── Load Balancer (HAProxy)
    │
    ├─── Management Network (VLAN 100)
    │       ├─── Proxmox VE Nodes (3+)
    │       ├─── Proxmox Backup Server
    │       └─── Monitoring Stack
    │
    └─── VM Production Network (VLAN 200)
            └─── Application VMs
```

## Key Design Decisions

### High Availability

- Minimum 3 Proxmox nodes for cluster quorum
- Shared storage (Ceph, NFS, or SAN) for live migration
- Automated failover with fencing
- Load balancer redundancy with Keepalived

### Backup Strategy (3-2-1 Rule)

- **3 copies**: Production + Local PBS + Remote/Cloud
- **2 media types**: Local storage + Cloud/Remote site
- **1 offsite copy**: Cloud storage or remote datacenter
- Automated verification with test restores

### Security Layers

1. Perimeter firewall (FortiGate)
2. Network segmentation (VLANs)
3. Host-based firewall (Proxmox)
4. Access control (VPN/2FA)
5. Encryption (TLS/SSH)
6. Audit logging

### Monitoring Philosophy

- Proactive monitoring prevents outages
- Multi-layer visibility (infrastructure to application)
- Automated alerting with escalation
- Historical data for capacity planning
- Security event correlation

## Deployment Checklist

### Phase 1: Planning

- [ ] Hardware compatibility verification
- [ ] Network architecture design
- [ ] IP allocation planning
- [ ] Storage sizing and RAID planning
- [ ] Licensing and subscription procurement

### Phase 2: Base Installation

- [ ] Proxmox VE installation on all nodes
- [ ] Network configuration (VLANs, bonds)
- [ ] Storage configuration (ZFS/LVM)
- [ ] Cluster creation and quorum setup
- [ ] PBS installation and configuration

### Phase 3: Security Hardening

- [ ] Firewall rules implementation
- [ ] VPN gateway deployment
- [ ] SSL certificate installation
- [ ] 2FA configuration
- [ ] Access control setup

### Phase 4: Backup Configuration

- [ ] PBS datastore creation
- [ ] Backup job scheduling
- [ ] Retention policy configuration
- [ ] Offsite replication setup
- [ ] Verification job automation

### Phase 5: Monitoring Deployment

- [ ] Prometheus and Grafana installation
- [ ] Exporter configuration on all nodes
- [ ] Alert rules creation
- [ ] Dashboard customization
- [ ] Log aggregation setup

### Phase 6: Testing & Validation

- [ ] Backup and restore testing
- [ ] Failover scenario testing
- [ ] Performance benchmarking
- [ ] Security penetration testing
- [ ] Documentation completion

## Maintenance Schedule

### Daily

- Monitor backup job completion
- Review critical alerts
- Check cluster health status

### Weekly

- Verify backup integrity
- Review performance metrics
- Security log analysis

### Monthly

- Test restore procedures
- Update security patches
- Review capacity trends
- Certificate expiration check

### Quarterly

- Disaster recovery drill
- Performance optimization review
- Security audit
- Documentation updates

## Resource Requirements

### Minimum Production Setup

- **3x Proxmox Nodes**: 32GB RAM, 8+ cores, 2x 480GB SSD each
- **1x PBS Server**: 16GB RAM, 4+ cores, 4TB+ storage
- **1x Monitoring Server**: 8GB RAM, 4 cores, 500GB storage
- **Network**: 10Gbps backbone recommended
- **Shared Storage**: 10TB+ usable (after RAID)

### Recommended Enterprise Setup

- **3-5x Proxmox Nodes**: 128GB+ RAM, 16+ cores each
- **2x PBS Servers**: 32GB RAM, 8TB+ storage each
- **Dedicated Monitoring**: 16GB RAM, 1TB storage
- **Network**: 25/40Gbps backbone
- **Enterprise SAN/Ceph**: 50TB+ usable

## Support and Resources

### Official Documentation

- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [Proxmox Backup Server Docs](https://pbs.proxmox.com/docs/)
- [Proxmox Forums](https://forum.proxmox.com/)

### Community Resources

- [r/Proxmox Reddit](https://reddit.com/r/proxmox)
- [Proxmox Discord](https://discord.gg/proxmox)
- [YouTube Tutorials](https://www.youtube.com/results?search_query=proxmox)

### Enterprise Support

- [Proxmox Subscription Plans](https://www.proxmox.com/en/proxmox-ve/pricing)
- Basic: Community support
- Standard: Business hours support
- Premium: 24/7 support with 2-hour response

## License

This documentation is provided as-is for educational and planning purposes. Always refer to official Proxmox documentation for the most current information.

## Contributing

To improve this documentation:

1. Test configurations in a lab environment first
2. Document any issues or improvements
3. Share feedback with the infrastructure team
4. Keep security considerations paramount

## Version History

- **v1.0** (2024-01): Initial production deployment guide
- Created comprehensive documentation for Proxmox production setup
- Includes security, backup, monitoring, and VPN configurations

---

_Last Updated: January 2024_
_Maintained by: Infrastructure Team_
