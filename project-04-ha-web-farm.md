# Project 4: High-Availability Web Farm with Keepalived + HAProxy

## Goal

Build a redundant load-balanced web service using Keepalived (VRRP) for virtual IP failover and HAProxy for traffic distribution.

## Skills Covered

| CCNA | RHEL |
|------|------|
| First-hop redundancy (HSRP/VRRP) | `keepalived` configuration |
| Load balancing concepts | `haproxy` configuration |
| Server health checking | `httpd` (Apache) deployment |
| Virtual IP (VIP) management | `systemd` service dependencies |
| Failover testing | Log analysis (`journalctl`, `/var/log/haproxy.log`) |

---

## Topology

```
        Nutanix AHV Cluster
    ┌─────────────────────────────────────┐
    │                                     │
    │   VLAN200-WebFarm                   │
    │   192.168.200.0/24                  │
    │        │                            │
    │   ┌────┴────┐                       │
    │   │  VIP    │  192.168.200.100      │
    │   │ (floats)│                       │
    │   └────┬────┘                       │
    │        │                            │
    │   ┌────┴────┐     ┌─────────┐       │
    │   │  LB01   │     │  LB02   │       │
    │   │ .200.11 │     │ .200.12 │       │
    │   │ MASTER  │     │ BACKUP  │       │
    │   └────┬────┘     └────┬────┘       │
    │        │               │            │
    │        └───────┬───────┘            │
    │                │                    │
    │           ┌────┴────┐               │
    │           │ HAProxy │               │
    │           │  :80    │               │
    │           └────┬────┘               │
    │                │                    │
    │        ┌───────┼───────┐            │
    │        │       │       │            │
    │   ┌────┴──┐ ┌──┴───┐ ┌┴─────┐      │
    │   │ WEB01 │ │ WEB02│ │WEB03│      │
    │   │.200.21│ │.200.22│ │.200.23│    │
    │   └───────┘ └──────┘ └─────┘      │
    │                                     │
    │   ┌─────────┐                       │
    │   │ CLIENT  │                       │
    │   │ .200.50 │                       │
    │   └─────────┘                       │
    │                                     │
    └─────────────────────────────────────┘
```

---

## Prerequisites

- [ ] 5 RHEL VMs: `LB01`, `LB02`, `WEB01`, `WEB02`, `WEB03`, `CLIENT`
- [ ] All on Nutanix network `VLAN200-WebFarm` (192.168.200.0/24)
- [ ] Packages staged: `keepalived`, `haproxy`, `httpd`

---

## Step 1: Configure Web Servers (WEB01, WEB02, WEB03)

### 1.1 Install Apache

```bash
sudo dnf install -y httpd
```

### 1.2 Create Unique Test Page

On **WEB01**:
```bash
sudo hostnamectl set-hostname web01.lab.local
sudo nmcli con add type ethernet ifname ens3 con-name web \
    ipv4.method manual ipv4.addresses 192.168.200.21/24
sudo nmcli con up web

echo "<html><body><h1>WEB01</h1><p>Server: 192.168.200.21</p></body></html>" | \
    sudo tee /var/www/html/index.html

sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

On **WEB02**:
```bash
sudo hostnamectl set-hostname web02.lab.local
sudo nmcli con add type ethernet ifname ens3 con-name web \
    ipv4.method manual ipv4.addresses 192.168.200.22/24
sudo nmcli con up web

echo "<html><body><h1>WEB02</h1><p>Server: 192.168.200.22</p></body></html>" | \
    sudo tee /var/www/html/index.html

sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

On **WEB03**:
```bash
sudo hostnamectl set-hostname web03.lab.local
sudo nmcli con add type ethernet ifname ens3 con-name web \
    ipv4.method manual ipv4.addresses 192.168.200.23/24
sudo nmcli con up web

echo "<html><body><h1>WEB03</h1><p>Server: 192.168.200.23</p></body></html>" | \
    sudo tee /var/www/html/index.html

sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

**Verify each web server:**
```bash
curl -s http://192.168.200.21/ | grep "<h1>"
# Expected: <h1>WEB01</h1>
```

---

## Step 2: Configure Load Balancer 1 (LB01 — Master)

### 2.1 Network

```bash
sudo hostnamectl set-hostname lb01.lab.local
sudo nmcli con add type ethernet ifname ens3 con-name lb \
    ipv4.method manual ipv4.addresses 192.168.200.11/24
sudo nmcli con up lb
```

### 2.2 Install and Configure Keepalived

```bash
sudo dnf install -y keepalived

sudo tee /etc/keepalived/keepalived.conf << 'EOF'
vrrp_instance VI_1 {
    state MASTER
    interface ens3
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LabPass123
    }
    virtual_ipaddress {
        192.168.200.100/24
    }
}
EOF

sudo systemctl enable --now keepalived

# Verify VIP is assigned
ip addr show ens3 | grep 192.168.200.100
# Expected: inet 192.168.200.100/24 scope global secondary ens3
```

### 2.3 Install and Configure HAProxy

```bash
sudo dnf install -y haproxy

sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

sudo tee /etc/haproxy/haproxy.cfg << 'EOF'
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

frontend web_frontend
    bind *:80
    default_backend web_backend

backend web_backend
    balance roundrobin
    option httpchk GET /
    http-check expect status 200
    server web01 192.168.200.21:80 check
    server web02 192.168.200.22:80 check
    server web03 192.168.200.23:80 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats auth admin:LabPass123
EOF

# Enable rsyslog for HAProxy
sudo tee /etc/rsyslog.d/49-haproxy.conf << 'EOF'
local2.*    /var/log/haproxy.log
EOF

sudo systemctl restart rsyslog
sudo systemctl enable --now haproxy

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=8404/tcp
sudo firewall-cmd --reload
```

---

## Step 3: Configure Load Balancer 2 (LB02 — Backup)

### 3.1 Network

```bash
sudo hostnamectl set-hostname lb02.lab.local
sudo nmcli con add type ethernet ifname ens3 con-name lb \
    ipv4.method manual ipv4.addresses 192.168.200.12/24
sudo nmcli con up lb
```

### 3.2 Keepalived (Backup)

```bash
sudo dnf install -y keepalived

sudo tee /etc/keepalived/keepalived.conf << 'EOF'
vrrp_instance VI_1 {
    state BACKUP
    interface ens3
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LabPass123
    }
    virtual_ipaddress {
        192.168.200.100/24
    }
}
EOF

sudo systemctl enable --now keepalived

# Verify VIP is NOT here (it's on LB01)
ip addr show ens3 | grep 192.168.200.100
# Expected: no output (VIP only on master)
```

### 3.3 HAProxy (Identical to LB01)

Copy the exact same `haproxy.cfg` from LB01:

```bash
sudo dnf install -y haproxy
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

# Copy from LB01 or recreate identically
sudo tee /etc/haproxy/haproxy.cfg << 'EOF'
[ ... same as LB01 config above ... ]
EOF

sudo systemctl restart rsyslog
sudo systemctl enable --now haproxy

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=8404/tcp
sudo firewall-cmd --reload
```

---

## Step 4: Configure Client

```bash
sudo hostnamectl set-hostname client.lab.local
sudo nmcli con add type ethernet ifname ens3 con-name client \
    ipv4.method manual ipv4.addresses 192.168.200.50/24 \
    ipv4.gateway "" ipv4.dns ""
sudo nmcli con up client
```

---

## Step 5: Validation Tests

### Test 1: VIP on Master

```bash
# On LB01
ip addr show ens3 | grep 192.168.200.100
# Expected: inet 192.168.200.100/24

# On LB02
ip addr show ens3 | grep 192.168.200.100
# Expected: no output
```

### Test 2: Load Balancing

```bash
# From CLIENT, hit the VIP multiple times
for i in {1..9}; do
    curl -s http://192.168.200.100/ | grep "<h1>"
done

# Expected: alternating responses
# <h1>WEB01</h1>
# <h1>WEB02</h1>
# <h1>WEB03</h1>
# <h1>WEB01</h1>
# ... etc (round-robin)
```

### Test 3: Health Check

```bash
# Check HAProxy stats
curl -s --user admin:LabPass123 http://192.168.200.11:8404/stats

# Or CLI
sudo socat stdio /var/lib/haproxy/stats <<< "show stat"
# Expected: all servers show status UP
```

### Test 4: Failover — Stop Apache on WEB01

```bash
# On WEB01
sudo systemctl stop httpd

# From CLIENT, hit VIP again
for i in {1..6}; do
    curl -s http://192.168.200.100/ | grep "<h1>"
done

# Expected: only WEB02 and WEB03 in rotation
# <h1>WEB02</h1>
# <h1>WEB03</h1>
# <h1>WEB02</h1>
# ... no WEB01
```

### Test 5: Failover — Stop LB01 Entirely

```bash
# On LB01
sudo systemctl stop keepalived

# On LB02, verify VIP moved
ip addr show ens3 | grep 192.168.200.100
# Expected: inet 192.168.200.100/24 — now on LB02!

# From CLIENT, still works
curl -s http://192.168.200.100/ | grep "<h1>"
# Expected: web page loads (from remaining servers)
```

### Test 6: VRRP Advertisements

```bash
# On LB02, watch VRRP traffic
sudo tcpdump -i ens3 -n vrrp

# Expected: periodic VRRP advertisements from 192.168.200.11
# If LB01 stops, LB02 takes over after ~3 seconds
```

---

## CCNA Concepts Mapped

| CCNA Topic | How It's Demonstrated |
|-----------|----------------------|
| HSRP/VRRP | Keepalived `vrrp_instance` with virtual IP |
| Virtual IP (VIP) | 192.168.200.100 floats between LB01 and LB02 |
| Load balancing | HAProxy `roundrobin` algorithm |
| Health checking | `option httpchk GET /` probes web servers |
| Failover | Stop service → VIP moves to backup |
| Preemption | LB01 (higher priority) reclaims VIP when restored |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| VIP on both LBs | `virtual_router_id` must match; check multicast on Nutanix network |
| VIP on neither | `systemctl status keepalived`, check `journalctl -u keepalived` |
| HAProxy 503 | `sudo socat stdio /var/lib/haproxy/stats <<< "show stat"` — check server state |
| No load balancing | `balance roundrobin` in backend, `check` keyword present |
| Stats page blank | `stats enable`, `stats uri /stats`, port 8404 open in firewall |
| Failover slow | `advert_int 1` (default 1 second), `preempt` enabled by default |

---

## Cleanup

```bash
# Stop services
sudo systemctl stop keepalived haproxy httpd

# Remove VIP if stuck
sudo ip addr del 192.168.200.100/24 dev ens3

# Disable services
sudo systemctl disable keepalived haproxy httpd
```

---

## Next Steps

- Replace **roundrobin** with **leastconn** or **source** (sticky sessions)
- Add **SSL termination** at HAProxy with a self-signed cert
- Configure **Keepalived** to track HAProxy process (failover if HAProxy dies, not just if LB dies)
- Add a **second VIP** for active-active load balancing
