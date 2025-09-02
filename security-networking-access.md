# Proxmox Security, Networking & Access Configuration

## Network Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Internet / Public Network                    │
└────────────┬────────────────────────┬───────────────────────────┘
             │                        │
    ┌────────▼────────┐      ┌───────▼────────┐
    │   Firewall      │      │  VPN Gateway   │
    │  (pfSense/OPNs) │      │  (WireGuard)   │
    └────────┬────────┘      └───────┬────────┘
             │                        │
    ┌────────▼────────────────────────▼────────┐
    │          DMZ Network (VLAN 10)           │
    │  ┌─────────────┐  ┌─────────────────┐   │
    │  │ Reverse     │  │ Load Balancer   │   │
    │  │ Proxy       │  │ (HAProxy)       │   │
    │  │ (Nginx)     │  │                 │   │
    │  └─────────────┘  └─────────────────┘   │
    └───────────────────┬──────────────────────┘
                        │
    ┌───────────────────▼──────────────────────┐
    │      Management Network (VLAN 100)       │
    │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
    │  │ Proxmox  │  │ Proxmox  │  │  PBS   │ │
    │  │ Node 1   │  │ Node 2   │  │ Server │ │
    │  └──────────┘  └──────────┘  └────────┘ │
    └───────────────────┬──────────────────────┘
                        │
    ┌───────────────────▼──────────────────────┐
    │       VM Network (VLAN 200-299)          │
    │   Production VMs with Public Services    │
    └───────────────────────────────────────────┘
```

## Firewall Configuration

### Host Firewall (Proxmox Built-in)

#### Enable Firewall
```bash
# Enable cluster-wide firewall
pvesh set /cluster/firewall/options --enable 1

# Enable on specific node
pvesh set /nodes/pve1/firewall/options --enable 1
```

#### Management Access Rules
```bash
# /etc/pve/firewall/cluster.fw

[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[IPSET management_ips]
10.0.100.0/24  # Internal management network
192.168.1.100  # Admin workstation
203.0.113.50   # VPN exit IP

[RULES]
# Management Access
IN ACCEPT -source +management_ips -p tcp -dport 8006 -log nolog # Proxmox Web UI
IN ACCEPT -source +management_ips -p tcp -dport 22 -log nolog   # SSH
IN ACCEPT -source +management_ips -p tcp -dport 3128 -log nolog # SPICE Proxy

# Cluster Communication (between nodes)
IN ACCEPT -source 10.0.100.0/24 -p udp -dport 5404:5405 # Corosync
IN ACCEPT -source 10.0.100.0/24 -p tcp -dport 60000:60050 # Migration

# Monitoring
IN ACCEPT -source 10.0.100.20 -p tcp -dport 9100 # Prometheus node exporter
IN ACCEPT -source 10.0.100.20 -p tcp -dport 9221 # Proxmox exporter

# Block everything else
IN DROP -log nolog
```

#### VM Firewall Rules
```bash
# /etc/pve/firewall/100.fw (VM-specific)

[OPTIONS]
enable: 1
ndp: 1
dhcp: 1
policy_in: DROP
policy_out: ACCEPT
log_level_in: info

[RULES]
# Web services
IN ACCEPT -p tcp -dport 80 -log nolog
IN ACCEPT -p tcp -dport 443 -log nolog

# Application specific
IN ACCEPT -source 10.0.200.0/24 -p tcp -dport 3306 # MySQL from app servers
IN ACCEPT -source 10.0.100.0/24 -p tcp -dport 22   # SSH from management

# Health checks from load balancer
IN ACCEPT -source 10.0.10.10 -p tcp -dport 80
```

### Perimeter Firewall (pfSense/OPNsense)

#### Port Forwarding Rules
```yaml
HTTP/HTTPS to Load Balancer:
  WAN: 0.0.0.0:80 → DMZ: 10.0.10.10:80
  WAN: 0.0.0.0:443 → DMZ: 10.0.10.10:443

VPN Access:
  WAN: 0.0.0.0:51820 → VPN: 10.0.100.1:51820 (WireGuard)
  WAN: 0.0.0.0:1194 → VPN: 10.0.100.1:1194 (OpenVPN)

# NEVER expose Proxmox ports directly:
# Block: 8006, 8007, 3128, 5900-5999, 22
```

## VPN Access Configuration

### WireGuard VPN for Admin Access

#### Install on Proxmox Host
```bash
apt update && apt install wireguard

# Generate keys
wg genkey | tee /etc/wireguard/private.key
cat /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key

# Configure interface
cat > /etc/wireguard/wg0.conf <<EOF
[Interface]
Address = 10.99.0.1/24
PrivateKey = $(cat /etc/wireguard/private.key)
ListenPort = 51820

# Admin client 1
[Peer]
PublicKey = CLIENT_PUBLIC_KEY_1
AllowedIPs = 10.99.0.2/32

# Admin client 2
[Peer]
PublicKey = CLIENT_PUBLIC_KEY_2
AllowedIPs = 10.99.0.3/32
EOF

# Enable and start
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0

# Firewall rule for VPN access to Proxmox
iptables -A INPUT -s 10.99.0.0/24 -p tcp --dport 8006 -j ACCEPT
```

#### Client Configuration
```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.99.0.2/32
DNS = 10.0.100.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = public.ip.address:51820
AllowedIPs = 10.0.100.0/24, 10.0.200.0/24
PersistentKeepalive = 25
```

### OpenVPN Alternative
```bash
# Install OpenVPN Access Server (in VM)
wget -O openvpn-as.deb https://openvpn.net/...
dpkg -i openvpn-as.deb

# Configure for Proxmox access
/usr/local/openvpn_as/scripts/sacli --key "vpn.server.routing.private_network.0" \
  --value "10.0.100.0/24" ConfigPut
/usr/local/openvpn_as/scripts/sacli start
```

## Reverse Proxy Configuration

### Nginx Reverse Proxy for Proxmox

#### Never Expose Proxmox Directly!
```nginx
# /etc/nginx/sites-available/proxmox-proxy

upstream proxmox {
    server 10.0.100.10:8006;
    server 10.0.100.11:8006 backup;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name proxmox.company.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name proxmox.company.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/proxmox.company.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/proxmox.company.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    # IP Whitelisting
    allow 203.0.113.0/24;  # Office network
    allow 10.99.0.0/24;    # VPN network
    deny all;
    
    # Proxy Configuration
    location / {
        proxy_pass https://proxmox;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support for noVNC
        proxy_read_timeout 86400;
    }
}
```

### Traefik Alternative (Docker-based)
```yaml
# docker-compose.yml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./traefik.yml:/etc/traefik/traefik.yml
      - ./dynamic.yml:/etc/traefik/dynamic.yml
      - /var/run/docker.sock:/var/run/docker.sock
      - ./acme.json:/acme.json

# dynamic.yml
http:
  routers:
    proxmox:
      rule: "Host(`proxmox.company.com`)"
      service: proxmox
      tls:
        certResolver: letsencrypt
      middlewares:
        - ip-whitelist
        - security-headers
  
  services:
    proxmox:
      loadBalancer:
        servers:
          - url: "https://10.0.100.10:8006"
          - url: "https://10.0.100.11:8006"
  
  middlewares:
    ip-whitelist:
      ipWhitelist:
        sourceRange:
          - "203.0.113.0/24"
          - "10.99.0.0/24"
    
    security-headers:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        contentTypeNosniff: true
        browserXssFilter: true
```

## Load Balancer Configuration

### HAProxy for VM Services

```bash
# /etc/haproxy/haproxy.cfg

global
    maxconn 4096
    log 127.0.0.1 local0
    ssl-default-bind-ciphers ECDHE+AESGCM:ECDHE+AES256:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog

# Stats page
stats enable
stats uri /haproxy-stats
stats realm HAProxy\ Statistics
stats auth admin:secure_password

# Frontend for web services
frontend web_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/combined.pem
    redirect scheme https if !{ ssl_fc }
    
    # ACLs for routing
    acl app1_acl hdr(host) -i app1.company.com
    acl app2_acl hdr(host) -i app2.company.com
    
    # Use backend based on ACL
    use_backend app1_backend if app1_acl
    use_backend app2_backend if app2_acl
    default_backend default_backend

# Backend pools
backend app1_backend
    balance roundrobin
    option httpchk GET /health
    server vm101 10.0.200.101:80 check
    server vm102 10.0.200.102:80 check
    server vm103 10.0.200.103:80 check backup

backend app2_backend
    balance leastconn
    cookie SERVERID insert indirect nocache
    server vm201 10.0.200.201:80 check cookie vm201
    server vm202 10.0.200.202:80 check cookie vm202
```

### Keepalived for HA Load Balancer
```bash
# /etc/keepalived/keepalived.conf

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass SecurePass123
    }
    
    virtual_ipaddress {
        10.0.10.100/24 dev eth0
    }
    
    track_script {
        chk_haproxy
    }
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}
```

## SSL Certificate Management

### Let's Encrypt with Certbot

#### For Proxmox Host
```bash
# Install certbot
apt install certbot

# Get certificate (DNS challenge for internal hosts)
certbot certonly --manual --preferred-challenges dns \
  -d proxmox.company.com

# Install certificate in Proxmox
cat /etc/letsencrypt/live/proxmox.company.com/fullchain.pem \
    /etc/letsencrypt/live/proxmox.company.com/privkey.pem > \
    /etc/pve/local/pveproxy-ssl.pem

systemctl restart pveproxy

# Auto-renewal script
cat > /etc/cron.monthly/proxmox-cert-renew <<'EOF'
#!/bin/bash
certbot renew --quiet
if [ $? -eq 0 ]; then
    cat /etc/letsencrypt/live/proxmox.company.com/fullchain.pem \
        /etc/letsencrypt/live/proxmox.company.com/privkey.pem > \
        /etc/pve/local/pveproxy-ssl.pem
    systemctl restart pveproxy
fi
EOF
chmod +x /etc/cron.monthly/proxmox-cert-renew
```

#### For Web Applications (HTTP-01 Challenge)
```bash
# Using certbot with nginx
certbot --nginx -d app.company.com -d www.app.company.com

# Using acme.sh (alternative)
curl https://get.acme.sh | sh
~/.acme.sh/acme.sh --issue -d app.company.com -w /var/www/html

# Wildcard certificate (DNS-01)
certbot certonly --manual --preferred-challenges dns \
  -d "*.company.com" -d company.com
```

### Certificate Management in VMs
```bash
#!/bin/bash
# /usr/local/bin/deploy-certs.sh

# Deploy certificates to VMs
CERT_PATH="/etc/letsencrypt/live/company.com"
VMS="101 102 103"

for VM in $VMS; do
    # Copy certs to VM
    qm guest exec $VM -- mkdir -p /etc/ssl/certs
    cat $CERT_PATH/fullchain.pem | qm guest exec $VM -- tee /etc/ssl/certs/fullchain.pem
    cat $CERT_PATH/privkey.pem | qm guest exec $VM -- tee /etc/ssl/certs/privkey.pem
    
    # Restart services
    qm guest exec $VM -- systemctl restart nginx
done
```

## Port Mapping Strategy

### Standard Port Assignments
```yaml
Management Ports (Internal Only):
  8006: Proxmox VE Web UI
  8007: Proxmox Backup Server Web UI
  3128: SPICE Proxy
  22: SSH (change to non-standard port)
  5900-5999: VNC Consoles
  
Public-Facing Ports:
  80: HTTP (redirect to 443)
  443: HTTPS
  51820: WireGuard VPN
  
Internal Service Ports:
  3306: MySQL (VM network only)
  5432: PostgreSQL (VM network only)
  6379: Redis (VM network only)
  9090: Prometheus
  3000: Grafana
  
High Availability Ports:
  5404-5405: Corosync Cluster
  2224: PCS/Pacemaker
  3121: Pacemaker Remote
  21064: DLM
  60000-60050: Live Migration
```

### NAT and Port Forwarding
```bash
# iptables NAT configuration
# Forward external port to internal VM

# Enable IP forwarding
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Port forwarding rule
iptables -t nat -A PREROUTING -p tcp -d PUBLIC_IP --dport 8080 \
  -j DNAT --to-destination 10.0.200.101:80

iptables -t nat -A POSTROUTING -p tcp -d 10.0.200.101 --dport 80 \
  -j SNAT --to-source 10.0.100.10

# Save rules
iptables-save > /etc/iptables/rules.v4
```

## Security Best Practices

### Access Control Matrix
```yaml
Access Levels:
  Public Internet:
    - Port 443 → Load Balancer → Web VMs
    - Port 51820 → VPN Gateway
    - DENY all other ports
    
  VPN Users:
    - Proxmox Web UI (via reverse proxy)
    - SSH to management network
    - VM consoles
    
  Internal Network:
    - Full access to management ports
    - Direct VM access
    - Backup operations
    
  DMZ:
    - Only reverse proxy access to internal
    - No direct access to Proxmox
```

### Security Checklist
```bash
# 1. Change default ports
sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config

# 2. Disable root login
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# 3. Enable 2FA for Proxmox
pveum user modify root@pam -tfa type=totp

# 4. Regular security updates
cat > /etc/cron.weekly/security-updates <<'EOF'
#!/bin/bash
apt update
apt upgrade -y
pveam update
EOF

# 5. Audit logging
cat >> /etc/pve/datacenter.cfg <<EOF
audit: 1
EOF

# 6. Session timeout
pvesh set /access/domains/pam --tfa oath --comment "30 min timeout" --timeout 1800
```

### Monitoring Access
```bash
# Monitor failed login attempts
journalctl -u pveproxy -f | grep "authentication failure"

# Track SSH access
tail -f /var/log/auth.log | grep sshd

# Monitor firewall drops
tail -f /var/log/pve-firewall.log | grep DROP
```

## Emergency Access Procedures

### Break Glass Account
```bash
# Create emergency admin account
pveum user add emergency@pve
pveum acl modify / --user emergency@pve --role Administrator

# Store credentials securely (encrypted)
echo "emergency:$(openssl rand -base64 32)" | \
  gpg --encrypt -r security-team > /root/emergency-creds.gpg

# Disable account by default
pveum user modify emergency@pve --enable 0

# Enable only when needed
pveum user modify emergency@pve --enable 1
```

This comprehensive security configuration ensures:
1. **Never expose Proxmox directly** - Always use VPN or reverse proxy
2. **Defense in depth** - Multiple security layers
3. **Proper network segmentation** - VLANs for isolation
4. **Automated certificate management** - Let's Encrypt integration
5. **High availability** - Load balancing and failover
6. **Audit trail** - Logging and monitoring all access