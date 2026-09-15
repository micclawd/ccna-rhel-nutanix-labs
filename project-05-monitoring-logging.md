# Project 5: Centralized Logging + Monitoring with Prometheus & Grafana

## Goal

Deploy a monitoring and logging stack on RHEL that collects metrics via Prometheus/node_exporter and aggregates logs via rsyslog — all fully air-gapped with no external dependencies.

## Skills Covered

| CCNA | RHEL |
|------|------|
| SNMP monitoring concepts | Prometheus server configuration |
| Syslog severity levels & facilities | `rsyslog` remote logging |
| Network device monitoring | `node_exporter` on all nodes |
| NetFlow/sFlow concepts | Grafana dashboard deployment |
| Centralized logging | Tarball installation (no containers) |

---

## Topology

```
        Nutanix AHV Cluster
    ┌─────────────────────────────────────────┐
    │                                         │
    │   VLAN300-Monitoring                    │
    │   192.168.250.0/24                      │
    │        │                                │
    │   ┌────┴────┐                           │
    │   │  MON01  │  192.168.250.10           │
    │   │         │                           │
    │   │ Prometheus    :9090                 │
    │   │ Grafana       :3000                 │
    │   │ rsyslog       :514 (UDP/TCP)        │
    │   │ Alertmanager  :9093 (optional)      │
    │   └────┬────┘                           │
    │        │                                │
    │   ┌────┴────┐  ┌─────────┐  ┌────────┐ │
    │   │ NODE01  │  │ NODE02  │  │ NODE03 │ │
    │   │.250.21  │  │.250.22  │  │.250.23 │ │
    │   │         │  │         │  │        │ │
    │   │node_exp │  │node_exp │  │node_exp│ │
    │   │rsyslog  │  │rsyslog  │  │rsyslog │ │
    │   └─────────┘  └─────────┘  └────────┘ │
    │                                         │
    └─────────────────────────────────────────┘
```

---

## Prerequisites

- [ ] 4 RHEL VMs: `MON01`, `NODE01`, `NODE02`, `NODE03`
- [ ] All on Nutanix network `VLAN300-Monitoring` (192.168.250.0/24)
- [ ] Tarballs staged on MON01 (transfer via ISO mount, SCP from jump host, or Nutanix file upload):
  - `prometheus-2.x.x.linux-amd64.tar.gz`
  - `node_exporter-1.x.x.linux-amd64.tar.gz`
  - `grafana-x.x.x.linux-amd64.tar.gz`

> **Air-gapped staging tip:** Download tarballs on a machine with internet, copy to the Nutanix cluster via Prism's "Upload File" feature or attach an ISO with the files.

---

## Step 1: Install Node Exporter on All Nodes

Run on **MON01**, **NODE01**, **NODE02**, and **NODE03**:

### 1.1 Extract and Install

```bash
# Create user
sudo useradd --no-create-home --shell /bin/false node_exporter

# Extract (adjust version to your tarball)
tar -xzf node_exporter-1.8.2.linux-amd64.tar.gz
sudo cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

# Clean up
rm -rf node_exporter-1.8.2.linux-amd64*
```

### 1.2 Create Systemd Service

```bash
sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter

# Verify
curl -s http://localhost:9100/metrics | head -5
# Expected: # HELP node_cpu_seconds_total ... etc
```

### 1.3 Open Firewall

```bash
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
```

---

## Step 2: Install Prometheus on MON01

### 2.1 Extract and Install

```bash
sudo useradd --no-create-home --shell /bin/false prometheus

tar -xzf prometheus-2.53.0.linux-amd64.tar.gz
sudo cp prometheus-2.53.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.53.0.linux-amd64/promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus

rm -rf prometheus-2.53.0.linux-amd64*
```

### 2.2 Configure Prometheus

```bash
sudo tee /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files: []

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets:
          - "192.168.250.10:9100"
          - "192.168.250.21:9100"
          - "192.168.250.22:9100"
          - "192.168.250.23:9100"
        labels:
          group: "production"
EOF

sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml

# Validate config
sudo -u prometheus promtool check config /etc/prometheus/prometheus.yml
# Expected: SUCCESS
```

### 2.3 Create Systemd Service

```bash
sudo tee /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus

sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload

# Verify
curl -s http://localhost:9090/-/healthy
# Expected: Prometheus Server is Healthy.
```

---

## Step 3: Install Grafana on MON01

### 3.1 Extract and Install

```bash
tar -xzf grafana-11.1.0.linux-amd64.tar.gz
sudo mv grafana-v11.1.0 /opt/grafana
sudo chown -R root:root /opt/grafana

# Create user
sudo useradd --no-create-home --shell /bin/false grafana
```

### 3.2 Create Systemd Service

```bash
sudo tee /etc/systemd/system/grafana.service << 'EOF'
[Unit]
Description=Grafana
Wants=network-online.target
After=network-online.target

[Service]
User=grafana
Group=grafana
Type=simple
ExecStart=/opt/grafana/bin/grafana-server \
    --config=/opt/grafana/conf/defaults.ini \
    --homepath=/opt/grafana

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now grafana

sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload

# Verify
curl -s http://localhost:3000/api/health
# Expected: {"commit":"...","database":"ok","version":"..."}
```

### 3.3 Configure Grafana Data Source

```bash
# Wait for Grafana to fully start (30 seconds)
sleep 30

# Add Prometheus as data source via API
curl -X POST \
    -H "Content-Type: application/json" \
    -d '{
        "name": "Prometheus",
        "type": "prometheus",
        "url": "http://localhost:9090",
        "access": "proxy",
        "isDefault": true
    }' \
    http://admin:admin@localhost:3000/api/datasources

# Expected: {"id":1,"message":"Datasource added",...}
```

> **Default Grafana credentials:** `admin` / `admin` — change on first login.

---

## Step 4: Configure Centralized Syslog on MON01

### 4.1 Configure rsyslog Server

```bash
sudo tee /etc/rsyslog.d/49-remote.conf << 'EOF'
# Load TCP and UDP reception
module(load="imtcp")
input(type="imtcp" port="514")

module(load="imudp")
input(type="imudp" port="514")

# Template for remote host logs
template(name="RemoteHostLog" type="string"
    string="/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log")

# Log everything from remote hosts
if $fromhost-ip != '127.0.0.1' then {
    action(type="omfile" dynaFile="RemoteHostLog")
    stop
}
EOF

sudo mkdir -p /var/log/remote

sudo systemctl restart rsyslog

sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --add-port=514/udp
sudo firewall-cmd --reload

# Verify listening
sudo ss -tlnp | grep :514
sudo ss -ulnp | grep :514
```

### 4.2 Configure Clients to Forward Logs

On **NODE01**, **NODE02**, **NODE03**:

```bash
sudo tee /etc/rsyslog.d/90-forward.conf << 'EOF'
# Forward all logs to MON01
*.* @192.168.250.10:514

# If MON01 is down, queue locally
$ActionQueueType LinkedList
$ActionQueueFileName srvrfwd
$ActionResumeRetryCount -1
$ActionQueueSaveOnShutdown on
EOF

sudo systemctl restart rsyslog

# Generate test log
logger "Test syslog from $(hostname)"
```

**Verify on MON01:**
```bash
sudo ls /var/log/remote/
# Expected: node01/ node02/ node03/

sudo cat /var/log/remote/node01/root.log
# Expected: "Test syslog from node01"
```

---

## Step 5: Validation Tests

### Test 1: Prometheus Targets

```bash
curl -s http://localhost:9090/api/v1/targets | \
    python3 -m json.tool | grep -E '"job"|"health"|"lastError"'

# Expected: all targets show "health": "up"
```

### Test 2: Node Metrics in Prometheus

```bash
# Query CPU usage
curl -s 'http://localhost:9090/api/v1/query?query=node_cpu_seconds_total' | \
    python3 -m json.tool | head -20

# Expected: JSON with metric data for each node
```

### Test 3: Grafana Dashboard

```bash
# Check if data source works
curl -s http://admin:admin@localhost:3000/api/datasources | \
    python3 -m json.tool | grep -E '"name"|"type"|"url"'

# Expected: Prometheus data source listed
```

**Manual verification:**
1. From a machine with browser access to the Nutanix network, open `http://192.168.250.10:3000`
2. Log in with `admin` / `admin`
3. Go to **Dashboards > New > Import**
4. Use dashboard ID **1860** (Node Exporter Full) — if Grafana can't reach the internet, manually create a dashboard with these queries:
   - CPU: `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
   - Memory: `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100`
   - Disk: `node_filesystem_avail_bytes / node_filesystem_size_bytes * 100`

### Test 4: Syslog Reception

```bash
# On MON01, check remote logs arrive in real time
sudo tail -f /var/log/remote/node01/*.log

# From NODE01, generate logs
sudo logger -p daemon.info "Daemon info test"
sudo logger -p auth.warn "Auth warning test"

# On MON01, verify they appear with correct facility
# Expected: messages in /var/log/remote/node01/daemon.log and /var/log/remote/node01/auth.log
```

### Test 5: Prometheus Alerting (Optional)

```bash
# Create a simple alert rule
sudo tee /etc/prometheus/alert.rules.yml << 'EOF'
groups:
  - name: node_alerts
    rules:
      - alert: HighCPU
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
EOF

# Add to prometheus.yml under rule_files:
sudo sed -i 's|rule_files: \[\]|rule_files:\n  - alert.rules.yml|' /etc/prometheus/prometheus.yml

# Reload
sudo systemctl reload prometheus
```

---

## CCNA Concepts Mapped

| CCNA Topic | How It's Demonstrated |
|-----------|----------------------|
| SNMP | Prometheus pull model (SNMP is push; conceptually similar) |
| Syslog | rsyslog facility/severity, remote logging, UDP 514 |
| Network monitoring | Node exporter = SNMP agent on steroids |
| Centralized management | MON01 = Cisco Prime / SolarWinds equivalent |
| Dashboards | Grafana = network operations center (NOC) view |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Prometheus target down | `curl http://<node>:9100/metrics` from MON01; check firewall |
| Grafana no data | `curl http://localhost:9090/api/v1/query?query=up` on MON01 |
| Syslog not arriving | `sudo tcpdump -i ens3 port 514` on MON01; check client forwarding config |
| Grafana won't start | `journalctl -u grafana -f`; check `/opt/grafana/data` permissions |
| Node exporter metrics missing | `systemctl status node_exporter`; check if process is running |

---

## Cleanup

```bash
# Stop services
sudo systemctl stop prometheus grafana node_exporter

# Remove binaries
sudo rm -f /usr/local/bin/prometheus /usr/local/bin/promtool /usr/local/bin/node_exporter

# Remove configs
sudo rm -rf /etc/prometheus /var/lib/prometheus /opt/grafana

# Remove syslog forwarding
sudo rm -f /etc/rsyslog.d/49-remote.conf /etc/rsyslog.d/90-forward.conf
sudo systemctl restart rsyslog
```

---

## Next Steps

- Add **Alertmanager** for email/webhook notifications (requires SMTP relay in air-gapped env)
- Deploy **blackbox_exporter** to probe HTTP/TCP endpoints from MON01
- Set up **Loki** for log aggregation (modern alternative to rsyslog)
- Create **Grafana dashboards** for each previous project (VPN status, HAProxy stats, etc.)
- Add **SNMP exporter** if you have physical network devices to monitor
