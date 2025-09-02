# FortiGate VPN Integration with Proxmox

## FortiGate Architecture with Proxmox

```
┌──────────────────────────────────────────────────────────────┐
│                        Internet                               │
└─────────────────────┬────────────────────────────────────────┘
                      │
         ┌────────────▼────────────┐
         │   FortiGate Firewall    │
         │   Model: 100F/200F+     │
         │   HA Active-Passive     │
         └────┬───────────────┬────┘
              │               │
    ┌─────────▼──────┐  ┌────▼───────────┐
    │  SSL VPN       │  │  IPSec VPN     │
    │  (FortiClient) │  │  (Site-to-Site)│
    └─────────┬──────┘  └────┬───────────┘
              │               │
    ┌─────────▼───────────────▼──────────┐
    │    FortiGate Internal Zones        │
    ├─────────────────────────────────────┤
    │ Management Zone (10.0.100.0/24)    │
    │ DMZ Zone (10.0.10.0/24)            │
    │ VM Production (10.0.200.0/24)      │
    │ Storage Zone (10.0.50.0/24)        │
    └─────────────────────────────────────┘
```

## FortiGate SSL VPN Configuration

### FortiGate SSL VPN Portal Setup

```bash
# FortiGate CLI Configuration
config vpn ssl settings
    set servercert "Fortinet_SSL"
    set tunnel-ip-pools "SSLVPN_TUNNEL_ADDR1"
    set tunnel-ipv6-pools "SSLVPN_TUNNEL_IPv6_ADDR1"
    set dns-server1 10.0.100.1
    set dns-server2 10.0.100.2
    set wins-server1 10.0.100.1
    set ipv6-dns-server1 2001:db8::1
    set source-interface "wan1"
    set source-address "all"
    set default-portal "proxmox-admins"
    set port 10443
    set dtls-tunnel enable
end

# Create IP Pool for VPN Clients
config firewall address
    edit "SSLVPN_TUNNEL_ADDR1"
        set type iprange
        set start-ip 10.99.1.1
        set end-ip 10.99.1.100
    next
end

# Create Admin Portal
config vpn ssl web portal
    edit "proxmox-admins"
        set tunnel-mode enable
        set ip-pools "SSLVPN_TUNNEL_ADDR1"
        set split-tunneling enable
        set split-tunneling-routing-address "proxmox_internal_networks"
        set web-mode enable
        set auto-connect enable
        set keep-alive enable
        set save-password enable
        set clipboard enable
        set skip-check-for-browser enable
        
        # Bookmarks for Proxmox access
        config bookmark group
            edit "Proxmox-Resources"
                config bookmarks
                    edit "Proxmox-Cluster"
                        set url "https://10.0.100.10:8006"
                        set description "Proxmox VE Cluster"
                        set sso disable
                    next
                    edit "Proxmox-Backup"
                        set url "https://10.0.100.20:8007"
                        set description "Proxmox Backup Server"
                    next
                    edit "Monitoring"
                        set url "https://10.0.100.30:3000"
                        set description "Grafana Dashboard"
                    next
                end
        end
    next
end
```

### User Authentication Configuration

```bash
# LDAP/Active Directory Integration
config user ldap
    edit "AD-SERVER"
        set server "10.0.100.5"
        set cnid "sAMAccountName"
        set dn "dc=company,dc=local"
        set type regular
        set username "ldap_bind@company.local"
        set password ENC <encrypted_password>
        set group-filter "(&(objectClass=group)(member=*))"
        set secure ldaps
        set port 636
    next
end

# Create User Groups
config user group
    edit "proxmox-admins"
        set member "AD-SERVER"
        match-group "CN=ProxmoxAdmins,OU=Groups,DC=company,DC=local"
    next
    edit "proxmox-operators"
        set member "AD-SERVER"
        match-group "CN=ProxmoxOperators,OU=Groups,DC=company,DC=local"
    next
end

# RADIUS Authentication (Alternative)
config user radius
    edit "RADIUS-MFA"
        set server "10.0.100.6"
        set secret ENC <encrypted_secret>
        set auth-type mschap-v2
        set nas-ip 10.0.100.1
        set timeout 10
    next
end
```

### FortiGate Firewall Policies for VPN

```bash
# VPN to Proxmox Management
config firewall policy
    edit 100
        set name "VPN-to-Proxmox-Mgmt"
        set srcintf "ssl.root"
        set dstintf "internal"
        set srcaddr "SSLVPN_TUNNEL_ADDR1"
        set dstaddr "Proxmox_Management_Servers"
        set action accept
        set schedule "always"
        set service "HTTPS" "SSH" "Proxmox-Ports"
        set utm-status enable
        set ssl-ssh-profile "certificate-inspection"
        set ips-sensor "default"
        set logtraffic all
        set nat enable
        set groups "proxmox-admins"
    next
end

# Custom Service Definitions
config firewall service custom
    edit "Proxmox-WebUI"
        set tcp-portrange 8006
    next
    edit "PBS-WebUI"
        set tcp-portrange 8007
    next
    edit "SPICE-Proxy"
        set tcp-portrange 3128
    next
    edit "VNC-Console"
        set tcp-portrange 5900-5999
    next
end

# Service Group for Proxmox
config firewall service group
    edit "Proxmox-Ports"
        set member "Proxmox-WebUI" "PBS-WebUI" "SPICE-Proxy" "VNC-Console"
    next
end
```

## FortiGate IPSec VPN (Site-to-Site)

### IPSec Configuration for Remote Proxmox Sites

```bash
# Phase 1 Configuration
config vpn ipsec phase1-interface
    edit "Proxmox-Site-B"
        set type static
        set interface "wan1"
        set ike-version 2
        set local-gw 203.0.113.1
        set remote-gw 203.0.113.2
        set psksecret ENC <pre-shared-key>
        set dpd on-idle
        set dpd-retryinterval 10
        set proposal aes256-sha256
        set dhgrp 14
        set nattraversal enable
        set fragmentation enable
        set dpd-retrycount 3
        set network-overlay enable
        set network-id 100
    next
end

# Phase 2 Configuration
config vpn ipsec phase2-interface
    edit "Proxmox-Site-B-P2"
        set phase1name "Proxmox-Site-B"
        set proposal aes256-sha256
        set pfs enable
        set dhgrp 14
        set auto-negotiate enable
        set src-subnet 10.0.100.0 255.255.255.0
        set dst-subnet 10.1.100.0 255.255.255.0
    next
end

# Static Routes for Remote Site
config router static
    edit 10
        set dst 10.1.100.0 255.255.255.0
        set device "Proxmox-Site-B"
        set comment "Remote Proxmox Management Network"
    next
end
```

## FortiClient Configuration

### FortiClient EMS (Endpoint Management)

```xml
<!-- FortiClient Configuration Profile -->
<forticlient_configuration>
    <vpn>
        <sslvpn>
            <connection>
                <name>Proxmox Management VPN</name>
                <server>vpn.company.com:10443</server>
                <username></username>
                <save_password>0</save_password>
                <certificate>Use System Certificate</certificate>
                <warn_invalid_cert>1</warn_invalid_cert>
                <prompt_certificate>0</prompt_certificate>
                <sso_enabled>0</sso_enabled>
                <use_external_browser>0</use_external_browser>
                <dual_stack>0</dual_stack>
                <dtls>1</dtls>
                <keep_alive>1</keep_alive>
            </connection>
        </sslvpn>
    </vpn>
    
    <endpoint_control>
        <enabled>1</enabled>
        <system_compliance>
            <antivirus>1</antivirus>
            <firewall>1</firewall>
            <min_os_version>Windows 10</min_os_version>
        </system_compliance>
    </endpoint_control>
</forticlient_configuration>
```

### Zero Trust Network Access (ZTNA)

```bash
# FortiGate ZTNA Configuration
config firewall access-proxy
    edit "Proxmox-ZTNA"
        set vip "Proxmox-VIP"
        set client-cert enable
        set log-blocked-traffic enable
        
        config api-gateway
            edit 1
                set url-map "/proxmox"
                set service "Proxmox-WebUI"
                config realservers
                    edit 1
                        set ip 10.0.100.10
                        set port 8006
                        set health-check enable
                    next
                    edit 2
                        set ip 10.0.100.11
                        set port 8006
                        set health-check enable
                    next
                end
            next
        end
    next
end

# ZTNA Policy
config firewall proxy-policy
    edit 1
        set name "ZTNA-Proxmox-Access"
        set proxy access-proxy
        set access-proxy "Proxmox-ZTNA"
        set srcintf "wan1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set utm-status enable
        set ztna-ems-tag "Proxmox_Admins_Devices"
        set logtraffic all
    next
end
```

## FortiGate High Availability for VPN

### HA Configuration

```bash
# Primary FortiGate
config system ha
    set group-id 10
    set group-name "FG-HA-Proxmox"
    set mode a-p
    set password ENC <encrypted_password>
    set hbdev "port3" 50 "port4" 50
    set session-pickup enable
    set session-pickup-connectionless enable
    set ha-mgmt-status enable
    
    config ha-mgmt-interfaces
        edit 1
            set interface "mgmt"
            set dst 0.0.0.0 0.0.0.0
            set gateway 10.0.100.254
        next
    end
    
    set override disable
    set priority 200
    set unicast-hb enable
    set unicast-hb-peerip 10.0.100.12
end

# Secondary FortiGate
config system ha
    set group-id 10
    set group-name "FG-HA-Proxmox"
    set mode a-p
    set password ENC <encrypted_password>
    set hbdev "port3" 50 "port4" 50
    set session-pickup enable
    set session-pickup-connectionless enable
    set ha-mgmt-status enable
    
    config ha-mgmt-interfaces
        edit 1
            set interface "mgmt"
            set dst 0.0.0.0 0.0.0.0
            set gateway 10.0.100.254
        next
    end
    
    set override disable
    set priority 100
    set unicast-hb enable
    set unicast-hb-peerip 10.0.100.11
end
```

## Integration with Proxmox

### Proxmox Firewall Rules for FortiGate VPN

```bash
# /etc/pve/firewall/cluster.fw

[IPSET fortigate_vpn_pool]
10.99.1.0/24  # FortiGate SSL VPN Pool

[IPSET fortigate_ipsec]
10.1.100.0/24  # Remote Site B

[RULES]
# Allow FortiGate VPN Users
IN ACCEPT -source +fortigate_vpn_pool -p tcp -dport 8006 -log nolog # Proxmox WebUI
IN ACCEPT -source +fortigate_vpn_pool -p tcp -dport 22 -log nolog   # SSH
IN ACCEPT -source +fortigate_vpn_pool -p tcp -dport 3128 -log nolog # SPICE
IN ACCEPT -source +fortigate_vpn_pool -p tcp -dport 5900:5999 -log nolog # VNC

# Allow IPSec Site-to-Site
IN ACCEPT -source +fortigate_ipsec -p tcp -dport 8006 -log nolog
IN ACCEPT -source +fortigate_ipsec -p tcp -dport 8007 -log nolog
```

### RADIUS Authentication for Proxmox

```bash
# Install RADIUS plugin
apt-get install libpve-access-control libauthen-pam-perl libnet-ldap-perl

# Configure RADIUS realm
pveum realm add radius-fortinet \
  --type pam \
  --server1 10.0.100.1 \
  --port 1812 \
  --secret 'radius_secret' \
  --comment "FortiGate RADIUS Authentication"

# Map RADIUS groups to Proxmox roles
pveum group add proxmox-admins -comment "FortiGate VPN Admins"
pveum acl modify / -group proxmox-admins -role Administrator
```

## FortiGate Monitoring & Logging

### FortiAnalyzer Integration

```bash
# Configure FortiGate to send logs
config log fortianalyzer setting
    set status enable
    set server "10.0.100.50"
    set source-ip 10.0.100.1
    set upload-option realtime
    set reliable enable
end

# Log VPN events
config log eventfilter
    set vpn enable
    set user enable
    set forward-traffic enable
end
```

### Syslog to Proxmox Monitoring

```bash
# FortiGate syslog configuration
config log syslogd setting
    set status enable
    set server "10.0.100.60"
    set port 514
    set facility local7
    set source-ip 10.0.100.1
    set format rfc5424
end

# On Proxmox monitoring server (rsyslog)
cat > /etc/rsyslog.d/30-fortigate.conf <<EOF
# FortiGate VPN Logs
:fromhost-ip, isequal, "10.0.100.1" /var/log/fortigate/vpn.log
& stop

# Parse FortiGate logs
template(name="FortiGateFormat" type="string"
  string="%timestamp% %hostname% %msg%\n")

# VPN connection alerts
if $fromhost-ip == '10.0.100.1' and $msg contains 'SSL VPN' then {
    action(type="omfile" file="/var/log/fortigate/ssl-vpn.log" template="FortiGateFormat")
}
EOF
```

## Security Best Practices

### FortiGate VPN Hardening

```bash
# 1. Enable Two-Factor Authentication
config user setting
    set auth-type fortitoken
    set auth-two-factor fortitoken
    set auth-timeout 300
end

# 2. Restrict VPN Access Hours
config firewall schedule recurring
    edit "Business-Hours"
        set start 07:00
        set end 19:00
        set day monday tuesday wednesday thursday friday
    next
end

# 3. Enable Anti-Replay
config vpn ssl settings
    set anti-replay strict
    set ssl-min-proto-ver tls1-2
    set banned-cipher RSA DES 3DES
end

# 4. Session Timeout
config vpn ssl settings
    set idle-timeout 1800
    set auth-timeout 28800
    set login-attempt 3
    set login-block-time 300
end

# 5. Host Checking
config vpn ssl web host-check-software
    edit "AV-Check"
        set type av
        set guid "Windows-Defender"
        set version ">= 4.18"
    next
end
```

### Monitoring Dashboard Queries

```sql
-- FortiAnalyzer SQL queries for Proxmox VPN monitoring

-- Active VPN Sessions
SELECT username, srcip, dstip, duration, bandwidth 
FROM vpn_sessions 
WHERE service='SSL-VPN' 
  AND dstip LIKE '10.0.100.%'
  AND status='connected'
ORDER BY duration DESC;

-- Failed Login Attempts
SELECT timestamp, username, srcip, reason 
FROM auth_logs 
WHERE action='failed' 
  AND service='SSL-VPN'
  AND timestamp > NOW() - INTERVAL 24 HOUR
ORDER BY timestamp DESC;

-- Bandwidth Usage by User
SELECT username, 
       SUM(sentbyte)/1048576 as sent_mb,
       SUM(rcvdbyte)/1048576 as recv_mb
FROM traffic_logs
WHERE policyid IN (100,101,102)
  AND timestamp > NOW() - INTERVAL 7 DAY
GROUP BY username
ORDER BY sent_mb DESC;
```

## Disaster Recovery

### VPN Backup Configuration

```bash
# Backup FortiGate configuration
execute backup config tftp fortigate-backup.conf 10.0.100.100

# Automated backup script on Proxmox
cat > /opt/backup-fortigate.sh <<'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR="/backup/fortigate"

# SSH to FortiGate and backup
ssh admin@10.0.100.1 "execute backup config tftp fg-backup-${DATE}.conf 10.0.100.10"

# Verify backup
if [ -f "${BACKUP_DIR}/fg-backup-${DATE}.conf" ]; then
    echo "Backup successful"
    # Keep only last 30 days
    find ${BACKUP_DIR} -name "fg-backup-*.conf" -mtime +30 -delete
else
    echo "Backup failed" | mail -s "FortiGate Backup Failed" admin@company.com
fi
EOF

# Add to crontab
echo "0 2 * * * /opt/backup-fortigate.sh" >> /etc/crontab
```

This FortiGate configuration provides:
1. **Enterprise SSL VPN** with FortiClient
2. **IPSec site-to-site** for multi-site Proxmox clusters
3. **ZTNA** for zero-trust access
4. **HA failover** for VPN redundancy
5. **Integration** with AD/LDAP and RADIUS
6. **Comprehensive logging** via FortiAnalyzer
7. **Security hardening** with 2FA and host checking