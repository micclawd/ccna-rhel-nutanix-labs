# Project 5: Centralized Logging + Monitoring with Prometheus & Grafana

**No Prism admin required.** Runs on your existing VMs on the flat network — every component installs from staged tarballs, nothing touches Prism or the internet.

## Goal

Build a NOC-in-a-box: Prometheus scrapes metrics from every node, rsyslog aggregates logs centrally, Grafana visualizes it — all offline.

## Skills Covered

| CCNA | RHEL |
|------|------|
| Syslog facilities, severities, UDP/TCP 514 | `rsyslog` remote templates & forwarding |
| SNMP-style device polling | Prometheus scrape model + `node_exporter` |
| Network monitoring / NOC dashboards | Grafana with offline dashboard provisioning |
| Alerting concepts | Prometheus alert rules |
| Service verification | `ss`, `curl` API checks, `journalctl` |

---

## Topology

```
              Existing flat network
    ┌────────────────────────────────────────────────┐
    │                                                │
    │   ┌─────────────────┐                          │
    │   │     MON01       │                          │
    │   │   (flat IP)     │                          │
    │   │                 │                          │
    │   │ Prometheus :9090│◄── scrapes ──┐           │
    │   │ Grafana    :3000│              │           │
    │   │ rsyslog    :514 │◄── logs ─┐   │           │
    │   └─────────────────┘          │   │           │
    │                                │   │           │
    │   ┌──────────┐  ┌──────────┐   │   │           │
    │   │ NODE01   │  │ NODE02   │   │   │           │
    │   │ (flat IP)│  │ (flat IP)│   │   │           │
    │   │ node_exp │  │ node_exp │───┼───┘           │
    │   │  :9100   │  │  :9100   │   │               │
    │   │ rsyslog  │  │ rsyslog  │───┘               │
    │   └──────────┘  └──────────┘                   │
    │                                                │
    └────────────────────────────────────────────────┘
```

---

## Prerequisites

- [ ] 2-4 existing RHEL VMs (MON01 + 1-3 nodes), sudo on all
- [ ] Staged tarballs transferred to the VMs (SCP from a jump host, an ISO with the files attached, or whatever file path your environment allows):
  - `prometheus-2.*.linux-amd64.tar.gz`
  - `node_exporter-1.*.linux-amd64.tar.gz`
  - `grafana-*.linux-amd64.tar.gz`
- [ ] On an internet-connected machine (for the Grafana dashboard JSON): download **Node Exporter Full** from grafana.com/dashboards/1860 → "Download JSON", and stage it alongside the tarballs

> Version numbers below are examples — adjust filenames to whatever you staged.

---

## Step 0: Discovery (all VMs)

```bash
FLAT_IP=$(ip -4 -o addr show scope global | awk 'NR==1{split($4,a,"/");print a[1]}')
echo "$FLAT_IP"
# Record MON01_IP and each NODE IP — you'll type them into prometheus.yml
```

---

## Step 1: node_exporter on EVERY VM (including MON01)

```bash
sudo useradd --no-create-home --shell /sbin/nologin node_exporter 2>/dev/null

tar -xzf node_exporter-1.*.linux-amd64.tar.gz
sudo cp node_exporter-1.*.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm -rf node_exporter-1.*.linux-amd64*

sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network-online.target
Wants=network-online.target

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
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload

curl -s http://localhost:9100/metrics | head -3
# Expected: # HELP ... / # TYPE ... / metric lines
```

---

## Step 2: Prometheus on MON01

### 2.1 Install

```bash
sudo useradd --no-create-home --shell /sbin/nologin prometheus 2>/dev/null

tar -xzf prometheus-2.*.linux-amd64.tar.gz
sudo cp prometheus-2.*.linux-amd64/{prometheus,promtool} /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}

sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
rm -rf prometheus-2.*.linux-amd64*
```

### 2.2 Config

```bash
sudo tee /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - alert.rules.yml

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets:
          - "MON01_IP:9100"
          - "NODE01_IP:9100"
          - "NODE02_IP:9100"
        labels:
          group: "lab"
EOF

# Replace MON01_IP / NODE01_IP / NODE02_IP with the flat IPs from Step 0
# (delete lines for nodes you don't have)

sudo tee /etc/prometheus/alert.rules.yml << 'EOF'
groups:
  - name: node_alerts
    rules:
      - alert: HighCPU
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} unreachable"
EOF

sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml /etc/prometheus/alert.rules.yml

sudo -u prometheus promtool check config /etc/prometheus/prometheus.yml
# Expected: SUCCESS: ... 0 errors
```

### 2.3 Service + Firewall

```bash
sudo tee /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus
After=network-online.target
Wants=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload

curl -s http://localhost:9090/-/healthy
# Expected: Prometheus Server is Healthy.
```

---

## Step 3: Grafana on MON01

### 3.1 Install

```bash
sudo useradd --no-create-home --shell /sbin/nologin grafana 2>/dev/null

tar -xzf grafana-*.linux-amd64.tar.gz
sudo mv grafana-v* /opt/grafana

sudo mkdir -p /var/lib/grafana/data /var/lib/grafana/dashboards
sudo chown -R grafana:grafana /opt/grafana /var/lib/grafana
```

### 3.2 Offline Dashboard Provisioning (no grafana.com needed)

```bash
# Drop the staged Node Exporter Full JSON here
sudo cp /path/to/node-exporter-full.json /var/lib/grafana/dashboards/
sudo chown grafana:grafana /var/lib/grafana/dashboards/*.json

# Tell Grafana to auto-load dashboards from that directory
sudo mkdir -p /opt/grafana/conf/provisioning/dashboards
sudo tee /opt/grafana/conf/provisioning/dashboards/local.yaml << 'EOF'
apiVersion: 1
providers:
  - name: local
    folder: ''
    type: file
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
EOF

# Provision the Prometheus data source offline too
sudo mkdir -p /opt/grafana/conf/provisioning/datasources
sudo tee /opt/grafana/conf/provisioning/datasources/prometheus.yaml << 'EOF'
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
EOF

sudo chown -R grafana:grafana /opt/grafana/conf/provisioning
```

### 3.3 Service + Firewall

```bash
sudo tee /etc/systemd/system/grafana.service << 'EOF'
[Unit]
Description=Grafana
After=network-online.target
Wants=network-online.target

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

sleep 10
curl -s http://localhost:3000/api/health
# Expected: {"database":"ok",...}
```

**Default login:** `admin` / `admin` (change on first login). The **Node Exporter Full** dashboard is already present under Dashboards — no internet needed.

---

## Step 4: Centralized Syslog

### 4.1 MON01 = Log Collector

```bash
sudo tee /etc/rsyslog.d/49-remote.conf << 'EOF'
module(load="imtcp")
input(type="imtcp" port="514")

module(load="imudp")
input(type="imudp" port="514")

template(name="RemoteHostLog" type="string"
    string="/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log")

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

sudo ss -tlnp | grep :514 && sudo ss -ulnp | grep :514
# Expected: rsyslogd listening on both
```

### 4.2 Nodes Forward Everything

On **each node** (and optionally MON01 itself):

```bash
sudo tee /etc/rsyslog.d/90-forward.conf << 'EOF'
*.* @@MON01_IP:514
EOF
# Replace MON01_IP. '@@' = TCP, single '@' = UDP. TCP recommended.

sudo systemctl restart rsyslog

logger "syslog test from $(hostname)"
```

**Verify on MON01:**
```bash
sudo ls /var/log/remote/
# Expected: one directory per node hostname

sudo tail -2 /var/log/remote/*/*.log
# Expected: your "syslog test from ..." lines, filed by sender and program
```

---

## Step 5: Validation Tests

### Test 1: All Targets Up

```bash
curl -s http://localhost:9090/api/v1/targets | \
    python3 -c "import json,sys; d=json.load(sys.stdin); [print(t['scrapeUrl'], '→', t['health']) for t in d['data']['activeTargets']]"
# Expected: every target → up
```

### Test 2: Metrics Actually Flowing

```bash
curl -s 'http://localhost:9090/api/v1/query?query=up' | \
    python3 -c "import json,sys; [print(r['metric']['job'], r['metric']['instance'], '=', r['value'][1]) for r in json.load(sys.stdin)['data']['result']]"
# Expected: every instance = 1
```

### Test 3: Grafana Data Source + Dashboards (browser-free)

```bash
curl -s http://admin:admin@localhost:3000/api/datasources | \
    python3 -c "import json,sys; [print(d['name'], d['type'], d['url']) for d in json.load(sys.stdin)]"
# Expected: Prometheus prometheus http://localhost:9090

curl -s http://admin:admin@localhost:3000/api/search | \
    python3 -c "import json,sys; [print(d['title']) for d in json.load(sys.stdin)]"
# Expected: Node Exporter Full (provisioned from the local JSON)
```

### Test 4: End-to-End Dashboard Check

From any machine with a browser that can reach MON01's flat IP: `http://<MON01_IP>:3000` → Dashboards → Node Exporter Full → all panels show data for every node.

### Test 5: Fire an Alert

```bash
# On NODE01: burn CPU for 3 minutes
timeout 180 bash -c 'while :; do :; done' &

# On MON01: watch the alert fire (within ~2-3 min)
curl -s http://localhost:9090/api/v1/alerts | \
    python3 -c "import json,sys; [print(a['labels']['alertname'], a['state']) for a in json.load(sys.stdin)['data']['alerts']]"
# Expected: HighCPU pending → firing, on the stressed node
```

### Test 6: Syslog Severity Demo (CCNA's 0-7 scale)

```bash
# On NODE01
logger -p kern.emerg "EMERGENCY test (severity 0)"
logger -p auth.warn "AUTH warning test (severity 4)"
logger -p daemon.debug "DEBUG test (severity 7)"

# On MON01
sudo tail -1 /var/log/remote/node01/kern.log
sudo tail -1 /var/log/remote/node01/auth.log
sudo tail -1 /var/log/remote/node01/daemon.log
# Expected: each message filed by facility — the CCNA severity table in action
```

---

## CCNA Concepts Mapped

| CCNA Topic | Where You Did It |
|-----------|------------------|
| Syslog facilities/severities | Test 6 — kern/auth/daemon, severities 0-7 |
| Syslog over UDP vs TCP 514 | `@@` (TCP) vs `@` (UDP) in forwarding config |
| SNMP-style polling | Prometheus scrape ≈ SNMP GET (pull model) |
| NOC dashboards | Grafana Node Exporter Full |
| Threshold alerting | `HighCPU` / `NodeDown` alert rules |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Target down in Prometheus | From MON01: `curl http://<node>:9100/metrics`. Firewall on node? `sudo firewall-cmd --list-all` |
| Grafana 502 / won't start | `journalctl -u grafana -n 30`. Permissions: `/var/lib/grafana` and `/opt/grafana` owned by grafana? |
| Dashboard empty but targets up | Data source provisioned? Step 5 Test 3. Time range top-right set to "Last 15 minutes"? |
| No remote logs | On MON01: `sudo tcpdump -i $DEV port 514 -c 4` while running `logger` on a node. No packets = forwarding config wrong on the node |
| Grafana dashboards dir ignored | Provisioning YAML path must match exactly; `journalctl -u grafana \| grep -i provision` |
| Prometheus OOM on small VM | Reduce retention: add `--storage.tsdb.retention.time=7d` to ExecStart |

---

## Cleanup

```bash
sudo systemctl disable --now prometheus grafana node_exporter
sudo rm -rf /opt/grafana /var/lib/grafana /etc/prometheus /var/lib/prometheus
sudo rm -f /usr/local/bin/{prometheus,promtool,node_exporter}
sudo rm -f /etc/systemd/system/{prometheus,grafana,node_exporter}.service
sudo rm -f /etc/rsyslog.d/49-remote.conf /etc/rsyslog.d/90-forward.conf
sudo systemctl daemon-reload
sudo systemctl restart rsyslog
sudo firewall-cmd --permanent --remove-port=9090/tcp --remove-port=3000/tcp \
    --remove-port=9100/tcp --remove-port=514/tcp --remove-port=514/udp
sudo firewall-cmd --reload
```

---

## Next Steps

- **Alertmanager**: route `NodeDown` to a local webhook or mail relay
- **blackbox_exporter** on MON01: probe the Project 4 VIP and Project 2's DNS/FTP — alert when services die
- **Loki** for searchable logs (modern companion to your rsyslog archive)
- Revisit Projects 1-4 and **instrument them**: node_exporter everywhere, dashboards for the web farm, syslog from the routers
