# Proxmox Monitoring & Observability Stack

## Complete Monitoring Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     Monitoring Stack Overview                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Metrics Collection          Logs                   Tracing        │
│  ┌──────────────┐           ┌──────────────┐      ┌─────────────┐ │
│  │ Prometheus   │           │ Loki/ELK     │      │ Jaeger      │ │
│  └──────┬───────┘           └──────┬───────┘      └──────┬──────┘ │
│         │                          │                      │        │
│  ┌──────▼───────────────────────────▼──────────────────────▼─────┐ │
│  │                     Grafana Dashboard                          │ │
│  └─────────────────────────────┬──────────────────────────────────┘ │
│                                │                                    │
│  ┌─────────────────────────────▼──────────────────────────────────┐ │
│  │              AlertManager / PagerDuty / Slack                  │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Data Sources:                                                     │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌──────────┐         │
│  │ Proxmox   │ │    PBS    │ │    VMs    │ │ Network  │         │
│  │ Nodes     │ │  Backup   │ │ Services  │ │ Devices  │         │
│  └───────────┘ └───────────┘ └───────────┘ └──────────┘         │
└────────────────────────────────────────────────────────────────────┘
```

## Prometheus Stack Deployment

### Prometheus Server Configuration

```yaml
# /opt/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'proxmox-prod'
    datacenter: 'dc1'

# Alerting configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - localhost:9093

# Load rules
rule_files:
  - "alerts/*.yml"

# Scrape configurations
scrape_configs:
  # Proxmox VE Nodes
  - job_name: 'proxmox'
    static_configs:
      - targets:
        - 10.0.100.10:9221  # pve1
        - 10.0.100.11:9221  # pve2
        - 10.0.100.12:9221  # pve3
    metrics_path: /pve
    params:
      module: [default]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 10.0.100.10:9221

  # Node Exporter (System Metrics)
  - job_name: 'node'
    static_configs:
      - targets:
        - 10.0.100.10:9100
        - 10.0.100.11:9100
        - 10.0.100.12:9100
        labels:
          type: 'hypervisor'

  # Proxmox Backup Server
  - job_name: 'proxmox-backup'
    static_configs:
      - targets:
        - 10.0.100.20:9100
        labels:
          type: 'backup'

  # VM Service Discovery
  - job_name: 'vms'
    proxmox_sd_configs:
      - server: 'https://10.0.100.10:8006'
        username: 'prometheus@pve'
        password_file: '/etc/prometheus/pve_password'
        verify_ssl: false
    relabel_configs:
      - source_labels: [__meta_proxmox_vmid]
        target_label: vmid
      - source_labels: [__meta_proxmox_name]
        target_label: vm_name
      - source_labels: [__meta_proxmox_status]
        regex: running
        action: keep

  # SNMP Network Devices
  - job_name: 'snmp'
    static_configs:
      - targets:
        - 10.0.1.1  # FortiGate
        - 10.0.1.2  # Core Switch
    metrics_path: /snmp
    params:
      module: [if_mib]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 10.0.100.30:9116  # SNMP exporter
```

### Proxmox VE Exporter Setup

```bash
# Install PVE Exporter on each node
wget https://github.com/prometheus-pve/prometheus-pve-exporter/releases/latest/download/prometheus-pve-exporter
chmod +x prometheus-pve-exporter
mv prometheus-pve-exporter /usr/local/bin/

# Create systemd service
cat > /etc/systemd/system/pve-exporter.service <<EOF
[Unit]
Description=Prometheus PVE Exporter
After=network.target

[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/prometheus-pve-exporter \
  --config.file=/etc/prometheus/pve.yml \
  --web.listen-address=:9221

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# PVE Exporter config
cat > /etc/prometheus/pve.yml <<EOF
default:
  user: prometheus@pve
  password: "secure_password"
  verify_ssl: false
EOF

systemctl enable --now pve-exporter
```

## Alert Rules Configuration

### Critical Proxmox Alerts

```yaml
# /opt/prometheus/alerts/proxmox.yml
groups:
  - name: proxmox_critical
    interval: 30s
    rules:
      # Node Down
      - alert: ProxmoxNodeDown
        expr: up{job="proxmox"} == 0
        for: 2m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "Proxmox node {{ $labels.instance }} is down"
          description: "Proxmox node has been unreachable for more than 2 minutes"
          runbook_url: "https://wiki.company.com/runbooks/proxmox-node-down"

      # High CPU Usage
      - alert: ProxmoxHighCPU
        expr: (100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 90% (current value: {{ $value }}%)"

      # Memory Pressure
      - alert: ProxmoxMemoryPressure
        expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low memory on {{ $labels.instance }}"
          description: "Available memory is less than 10% (current: {{ $value }}%)"

      # Storage Space
      - alert: ProxmoxStorageSpace
        expr: (pve_storage_usage_bytes / pve_storage_total_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Storage {{ $labels.storage }} is almost full"
          description: "Storage usage is above 85% (current: {{ $value }}%)"

      # VM Down
      - alert: VMDown
        expr: pve_vm_status != 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "VM {{ $labels.name }} ({{ $labels.vmid }}) is not running"
          description: "VM has been down for more than 5 minutes"

      # Backup Failed
      - alert: BackupFailed
        expr: pve_backup_status != 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Backup failed for VM {{ $labels.vmid }}"
          description: "Backup job failed with status {{ $value }}"

      # Cluster Quorum Lost
      - alert: ClusterQuorumLost
        expr: pve_cluster_quorum == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Proxmox cluster lost quorum"
          description: "Cluster quorum has been lost - immediate action required"

      # Certificate Expiry
      - alert: CertificateExpiringSoon
        expr: (pve_cert_expiry_timestamp_seconds - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Certificate expiring soon on {{ $labels.instance }}"
          description: "Certificate will expire in {{ $value }} days"
```

## Grafana Dashboard Configuration

### Main Proxmox Dashboard JSON

```json
{
  "dashboard": {
    "title": "Proxmox Production Overview",
    "panels": [
      {
        "title": "Cluster Status",
        "gridPos": {"h": 8, "w": 6, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "up{job='proxmox'}",
            "legendFormat": "{{ instance }}"
          }
        ],
        "type": "stat"
      },
      {
        "title": "Total VMs Running",
        "gridPos": {"h": 8, "w": 6, "x": 6, "y": 0},
        "targets": [
          {
            "expr": "sum(pve_vm_status == 1)",
            "legendFormat": "Running VMs"
          }
        ],
        "type": "gauge"
      },
      {
        "title": "CPU Usage per Node",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "100 - (avg by (instance) (rate(node_cpu_seconds_total{mode='idle'}[5m])) * 100)",
            "legendFormat": "{{ instance }}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Memory Usage",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
        "targets": [
          {
            "expr": "(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100",
            "legendFormat": "{{ instance }}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Storage Usage",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8},
        "targets": [
          {
            "expr": "(pve_storage_usage_bytes / pve_storage_total_bytes) * 100",
            "legendFormat": "{{ storage }} on {{ instance }}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Network Traffic",
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 16},
        "targets": [
          {
            "expr": "rate(node_network_receive_bytes_total[5m])",
            "legendFormat": "RX {{ device }} on {{ instance }}"
          },
          {
            "expr": "rate(node_network_transmit_bytes_total[5m])",
            "legendFormat": "TX {{ device }} on {{ instance }}"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

### Deploy Grafana

```bash
# Docker Compose for Grafana
cat > /opt/monitoring/docker-compose.yml <<EOF
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alerts:/etc/prometheus/alerts
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=90d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel,grafana-worldmap-panel
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.company.com:587
      - GF_SMTP_USER=alerts@company.com
      - GF_SMTP_PASSWORD=smtp_password
    volumes:
      - grafana_data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    ports:
      - "9093:9093"
    restart: unless-stopped

  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    pid: "host"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:
EOF

docker-compose up -d
```

## Log Aggregation with Loki

### Loki Configuration

```yaml
# /opt/loki/loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2023-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    cache_ttl: 24h
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h
```

### Promtail Configuration (Log Shipper)

```yaml
# /opt/promtail/promtail-config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://10.0.100.30:3100/loki/api/v1/push

scrape_configs:
  # Proxmox system logs
  - job_name: proxmox_system
    static_configs:
      - targets:
          - localhost
        labels:
          job: proxmox
          host: pve1
          __path__: /var/log/pve/*.log

  # Proxmox task logs
  - job_name: proxmox_tasks
    static_configs:
      - targets:
          - localhost
        labels:
          job: proxmox_tasks
          host: pve1
          __path__: /var/log/pve/tasks/*

  # Syslog
  - job_name: syslog
    syslog:
      listen_address: 0.0.0.0:514
      labels:
        job: syslog
    relabel_configs:
      - source_labels: ['__syslog_message_hostname']
        target_label: 'host'

  # Journal logs
  - job_name: journal
    journal:
      path: /var/log/journal
      labels:
        job: systemd-journal
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'
      - source_labels: ['__journal__hostname']
        target_label: 'hostname'

  # Container logs
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containers
          __path__: /var/lib/lxc/*/rootfs/var/log/*.log
```

## ELK Stack Alternative

### Elasticsearch, Logstash, Kibana Setup

```yaml
# /opt/elk/docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=elastic_password
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.10.0
    container_name: logstash
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      - "LS_JAVA_OPTS=-Xmx1g -Xms1g"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.10.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=kibana_password
    ports:
      - "5601:5601"

volumes:
  es_data:
```

### Logstash Pipeline for Proxmox

```ruby
# /opt/elk/logstash.conf
input {
  # Syslog input
  syslog {
    port => 5514
    type => "proxmox-syslog"
  }

  # Filebeat input
  beats {
    port => 5044
    type => "proxmox-logs"
  }

  # Direct file input
  file {
    path => "/var/log/pve-firewall.log"
    start_position => "beginning"
    type => "firewall"
  }
}

filter {
  if [type] == "proxmox-syslog" {
    grok {
      match => {
        "message" => "%{SYSLOGTIMESTAMP:timestamp} %{SYSLOGHOST:hostname} %{DATA:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:msg}"
      }
    }
    
    date {
      match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }

  if [type] == "firewall" {
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:timestamp} %{WORD:vmid} %{WORD:action} %{WORD:direction}: %{GREEDYDATA:details}"
      }
    }
    
    kv {
      source => "details"
      field_split => " "
      value_split => "="
    }
  }

  # Parse Proxmox task logs
  if [path] =~ /\/tasks\// {
    grok {
      match => {
        "message" => "%{DATA:user}@%{DATA:realm}: %{WORD:status} %{GREEDYDATA:task_details}"
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    user => "elastic"
    password => "elastic_password"
    index => "proxmox-%{type}-%{+YYYY.MM.dd}"
  }

  # Alert on critical events
  if [level] == "ERROR" or [level] == "CRITICAL" {
    email {
      to => "ops-team@company.com"
      subject => "Proxmox Alert: %{level} on %{hostname}"
      body => "%{message}"
    }
  }
}
```

## Distributed Tracing with Jaeger

### Jaeger Deployment

```yaml
# /opt/jaeger/docker-compose.yml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    environment:
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "5775:5775/udp"
      - "6831:6831/udp"
      - "6832:6832/udp"
      - "5778:5778"
      - "16686:16686"  # UI
      - "14268:14268"
      - "14250:14250"
      - "9411:9411"
    restart: unless-stopped
```

## Alert Manager Configuration

### Multi-Channel Alerting

```yaml
# /opt/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_from: 'proxmox-alerts@company.com'
  smtp_smarthost: 'smtp.company.com:587'
  smtp_auth_username: 'alerts@company.com'
  smtp_auth_password: 'smtp_password'
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default-receiver'
  
  routes:
    - match:
        severity: critical
      receiver: 'critical-receiver'
      continue: true
      
    - match:
        severity: warning
      receiver: 'warning-receiver'
      
    - match:
        team: infrastructure
      receiver: 'infra-team'

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'ops-team@company.com'
        
  - name: 'critical-receiver'
    email_configs:
      - to: 'oncall@company.com'
        send_resolved: true
    slack_configs:
      - channel: '#critical-alerts'
        title: 'Critical Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    pagerduty_configs:
      - service_key: 'YOUR-PAGERDUTY-KEY'
        
  - name: 'warning-receiver'
    slack_configs:
      - channel: '#warnings'
        send_resolved: true
        
  - name: 'infra-team'
    email_configs:
      - to: 'infrastructure@company.com'
    webhook_configs:
      - url: 'http://10.0.100.40:8080/webhook'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

## Health Check & Synthetic Monitoring

### Blackbox Exporter Configuration

```yaml
# /opt/blackbox/blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200, 201, 202, 204]
      method: GET
      follow_redirects: true
      fail_if_ssl: false
      fail_if_not_ssl: false
      tls_config:
        insecure_skip_verify: true

  http_post_2xx:
    prober: http
    timeout: 5s
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"check": "health"}'

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s

  dns_check:
    prober: dns
    timeout: 5s
    dns:
      query_name: "proxmox.company.local"
      query_type: "A"

# Prometheus scrape config for blackbox
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://10.0.100.10:8006  # Proxmox UI
        - https://10.0.100.20:8007  # PBS UI
        - http://10.0.200.101/health  # App health endpoint
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 10.0.100.35:9115  # Blackbox exporter
```

## Performance Monitoring

### System Performance Metrics

```bash
#!/bin/bash
# /opt/monitoring/collect-performance.sh

# Collect ZFS metrics
cat > /opt/monitoring/zfs-exporter.sh <<'EOF'
#!/bin/bash
while true; do
  # ARC stats
  arc_size=$(awk '/^size/ {print $3}' /proc/spl/kstat/zfs/arcstats)
  arc_hit_ratio=$(awk '/^hits/ {h=$3} /^misses/ {m=$3} END {print h/(h+m)*100}' /proc/spl/kstat/zfs/arcstats)
  
  # Pool stats
  zpool list -Hp | while read pool size alloc free ckpoint expandsz frag cap dedup health altroot; do
    echo "zfs_pool_size_bytes{pool=\"$pool\"} $size"
    echo "zfs_pool_allocated_bytes{pool=\"$pool\"} $alloc"
    echo "zfs_pool_free_bytes{pool=\"$pool\"} $free"
    echo "zfs_pool_fragmentation_percent{pool=\"$pool\"} $frag"
  done
  
  sleep 15
done
EOF

# Collect Ceph metrics (if using Ceph)
cat > /opt/monitoring/ceph-exporter.sh <<'EOF'
#!/bin/bash
while true; do
  # Ceph status
  ceph_health=$(ceph health -f json | jq -r '.status')
  ceph_osd_up=$(ceph osd stat -f json | jq '.num_up_osds')
  ceph_osd_in=$(ceph osd stat -f json | jq '.num_in_osds')
  
  echo "ceph_health{status=\"$ceph_health\"} 1"
  echo "ceph_osd_up $ceph_osd_up"
  echo "ceph_osd_in $ceph_osd_in"
  
  # Pool stats
  ceph df -f json | jq -r '.pools[] | "ceph_pool_bytes{pool=\"\(.name)\"} \(.stats.bytes_used)"'
  
  sleep 30
done
EOF
```

## Custom Dashboards

### VM Performance Dashboard

```json
{
  "dashboard": {
    "title": "VM Performance Metrics",
    "panels": [
      {
        "title": "VM CPU Usage Top 10",
        "targets": [
          {
            "expr": "topk(10, pve_vm_cpu_usage_ratio * 100)",
            "legendFormat": "{{ name }} ({{ vmid }})"
          }
        ],
        "type": "bargauge"
      },
      {
        "title": "VM Memory Usage",
        "targets": [
          {
            "expr": "(pve_vm_memory_usage_bytes / pve_vm_memory_total_bytes) * 100",
            "legendFormat": "{{ name }}"
          }
        ],
        "type": "heatmap"
      },
      {
        "title": "VM Disk I/O",
        "targets": [
          {
            "expr": "rate(pve_vm_disk_read_bytes[5m])",
            "legendFormat": "Read {{ name }}"
          },
          {
            "expr": "rate(pve_vm_disk_write_bytes[5m])",
            "legendFormat": "Write {{ name }}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "VM Network Traffic",
        "targets": [
          {
            "expr": "rate(pve_vm_network_receive_bytes[5m])",
            "legendFormat": "RX {{ name }}"
          },
          {
            "expr": "rate(pve_vm_network_transmit_bytes[5m])",
            "legendFormat": "TX {{ name }}"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

## Monitoring Best Practices

### Key Metrics to Monitor

```yaml
Infrastructure Level:
  - CPU: Usage, load average, context switches
  - Memory: Usage, swap, cache, buffers
  - Storage: Usage, I/O latency, throughput, errors
  - Network: Bandwidth, errors, drops, latency
  - Temperature: CPU, disk, ambient

Proxmox Specific:
  - Cluster: Quorum status, node availability
  - VMs: Status, resource usage, migration status
  - Storage: Pool health, replication status
  - Backup: Job status, duration, size
  - Tasks: Failed tasks, long-running tasks

Application Level:
  - Response time: API latency, page load time
  - Error rate: 4xx, 5xx errors
  - Throughput: Requests per second
  - Business metrics: User activity, transactions

Security:
  - Failed logins: SSH, web UI, API
  - Firewall: Dropped packets, blocked IPs
  - Certificates: Expiration dates
  - Updates: Pending security patches
```

### Retention Policies

```yaml
Metrics Retention:
  Raw (15s): 7 days
  5-minute: 30 days
  1-hour: 90 days
  1-day: 1 year

Logs Retention:
  System logs: 30 days
  Security logs: 90 days
  Audit logs: 1 year
  Debug logs: 7 days

Backup Monitoring:
  Success/failure: 90 days
  Performance metrics: 30 days
  Detailed logs: 14 days
```

This comprehensive monitoring stack provides:
1. **Full observability** with metrics, logs, and traces
2. **Multi-layer monitoring** from infrastructure to application
3. **Intelligent alerting** with escalation and inhibition
4. **Performance tracking** for capacity planning
5. **Security monitoring** for compliance and threat detection
6. **Custom dashboards** for different stakeholder needs