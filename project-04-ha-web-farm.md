# Project 4: High-Availability Web Farm with Keepalived + HAProxy

**No Prism admin required.** Runs on your existing VMs on the flat network. The only network-level need: **one unused IP on your flat subnet** for the virtual IP (VIP) — plus unicast VRRP so you don't depend on multicast.

## Goal

Float a virtual IP between two load balancers with VRRP, and round-robin web traffic across backend servers — the RHEL equivalent of HSRP + a Cisco ACE/F5.

## Skills Covered

| CCNA | RHEL |
|------|------|
| HSRP/VRRP: virtual IP, priority, preemption | `keepalived` with unicast VRRP |
| Load balancing algorithms | `haproxy` roundrobin / leastconn |
| Server health checking | `option httpchk` + stats socket |
| Failover testing | `systemctl stop` → watch VIP migrate |
| TCP/UDP port behavior | firewalld service rules |

---

## Topology

```
              Existing flat network (whatever yours is — VIP lives here)
    ┌────────────────────────────────────────────────────┐
    │                                                    │
    │              VIP: <FREE_FLAT_IP>  (floats)         │
    │                       │                            │
    │        ┌──────────────┼──────────────┐             │
    │        │              │              │             │
    │   ┌────┴────┐    ┌────┴────┐         │             │
    │   │  LB01   │    │  LB02   │         │             │
    │   │ MASTER  │◄──►│ BACKUP  │  unicast VRRP         │
    │   │ prio150 │    │ prio100 │         │             │
    │   └────┬────┘    └────┬────┘         │             │
    │        │  HAProxy :80  │             │             │
    │        └───────┬───────┘             │             │
    │                │                     │             │
    │        ┌───────┴────────┐            │             │
    │        │                │            │             │
    │   ┌────┴────┐     ┌────┴────┐   ┌────┴────┐        │
    │   │  WEB01  │     │  WEB02  │   │ CLIENT  │        │
    │   │ (flat)  │     │ (flat)  │   │ (flat)  │        │
    │   └─────────┘     └─────────┘   └─────────┘        │
    └────────────────────────────────────────────────────┘
```

**Minimum footprint: 2 VMs** — run LB+WEB co-located on each (LB01+WEB01, LB02+WEB02). Every skill still demonstrated. The guide shows the 4-VM layout; co-located notes inline.

---

## Prerequisites

- [ ] 2-4 existing RHEL VMs, sudo on all
- [ ] Packages: `keepalived haproxy httpd` (install from mounted RHEL ISO repo — see Project 2 prerequisites for the ISO-repo pattern)
- [ ] **One free IP on your flat subnet** for the VIP

### Verify the VIP Candidate Is Free (MANDATORY)

```bash
DEV=$(ip route show default | awk '{print $5; exit}')

VIP_CANDIDATE=<A_HIGH_IP_ON_YOUR_SUBNET>   # e.g. if flat is 172.205.2.0/24, try 172.205.2.250

ping -c 2 "$VIP_CANDIDATE"
# Expected: 100% packet loss

sudo dnf install -y arping 2>/dev/null
sudo arping -c 2 -D -I "$DEV" "$VIP_CANDIDATE"
# Expected: 0 responses / 100% unanswered — NOBODY owns this IP
```

> If either check gets a reply, pick another address. Claiming a live IP breaks someone else's VM and VRRP will fight them for it. Record your choice: `VIP=<verified-free-ip>`.

---

## Step 0: Discovery (all VMs)

```bash
FLAT_IP=$(ip -4 -o addr show scope global | awk 'NR==1{split($4,a,"/");print a[1]}')
DEV=$(ip route show default | awk '{print $5; exit}')
echo "$FLAT_IP on $DEV"
# Record: LB01_IP, LB02_IP, WEB01_IP, WEB02_IP — used throughout
```

---

## Step 1: Web Servers (WEB01, WEB02)

On **WEB01**:

```bash
sudo hostnamectl set-hostname web01.lab.local
sudo dnf install -y httpd

echo "<html><body><h1>WEB01</h1><p>Served from $(hostname)</p></body></html>" | \
    sudo tee /var/www/html/index.html

sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

curl -s http://localhost/ | grep h1
# Expected: <h1>WEB01</h1>
```

On **WEB02**: identical, but `hostname web02.lab.local` and `<h1>WEB02</h1>`.

> **2-VM variant:** run this web server on the LB VMs themselves; in the HAProxy config below, point the backends at the LBs' own flat IPs.

---

## Step 2: HAProxy (LB01 and LB02 — identical config)

```bash
sudo dnf install -y haproxy

sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

sudo tee /etc/haproxy/haproxy.cfg << EOF
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
    retries                 3
    timeout http-request    10s
    timeout connect         5s
    timeout client          30s
    timeout server          30s
    timeout check           5s

frontend web_frontend
    bind *:80
    default_backend web_backend

backend web_backend
    balance roundrobin
    option httpchk GET /
    server web01 <WEB01_IP>:80 check fall 2 rise 1
    server web02 <WEB02_IP>:80 check fall 2 rise 1

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
EOF
# Fill in WEB01_IP / WEB02_IP with the flat IPs from Step 0
```

### SELinux + Logging + Start

```bash
# HAProxy needs to make outbound HTTP connections to backends
sudo setsebool -P haproxy_connect_any 1

# Route HAProxy logs to a file
sudo tee /etc/rsyslog.d/49-haproxy.conf << 'EOF'
local2.*    /var/log/haproxy.log
EOF
sudo systemctl restart rsyslog

sudo systemctl enable --now haproxy
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=8404/tcp
sudo firewall-cmd --reload

# Backends healthy?
echo "show stat" | sudo socat stdio /var/lib/haproxy/stats | cut -d, -f1,2,18 | grep web_backend
# Expected: web_backend,web01,UP  and  web_backend,web02,UP
```

---

## Step 3: Keepalived with Unicast VRRP

Multicast VRRP may be filtered on a shared network. Unicast peering works everywhere.

### LB01 (MASTER)

```bash
sudo dnf install -y keepalived
sudo hostnamectl set-hostname lb01.lab.local

sudo tee /etc/keepalived/keepalived.conf << EOF
vrrp_instance VI_1 {
    state MASTER
    interface <DEV>
    unicast_src_ip <LB01_IP>
    unicast_peer {
        <LB02_IP>
    }
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LabPass1
    }
    virtual_ipaddress {
        <VIP>/32
    }
}
EOF
```

### LB02 (BACKUP)

```bash
sudo dnf install -y keepalived
sudo hostnamectl set-hostname lb02.lab.local

sudo tee /etc/keepalived/keepalived.conf << EOF
vrrp_instance VI_1 {
    state BACKUP
    interface <DEV>
    unicast_src_ip <LB02_IP>
    unicast_peer {
        <LB01_IP>
    }
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LabPass1
    }
    virtual_ipaddress {
        <VIP>/32
    }
}
EOF
```

> Replace `<DEV>` (from Step 0), `<LB01_IP>`, `<LB02_IP>`, `<VIP>` on each. `/32` on the VIP avoids the auto-generated subnet route — cleanest on a shared L2.

### Start Both

```bash
sudo systemctl enable --now keepalived

# On LB01 — VIP should be HERE
ip addr show "$DEV" | grep "$VIP"
# Expected: inet <VIP>/32 scope global ...

# On LB02 — should be ABSENT
ip addr show "$DEV" | grep "$VIP"
# Expected: no output
```

---

## Step 4: Validation Tests (from CLIENT, or any VM)

### Test 1: Load Balancing Round-Robin

```bash
for i in 1 2 3 4; do curl -s -m 5 "http://$VIP/" | grep h1; done
# Expected:
# <h1>WEB01</h1>
# <h1>WEB02</h1>
# <h1>WEB01</h1>
# <h1>WEB02</h1>
```

### Test 2: Health Check Ejects a Dead Server

```bash
# On WEB01
sudo systemctl stop httpd

# Back on the test VM
for i in 1 2 3 4; do curl -s -m 5 "http://$VIP/" | grep h1; done
# Expected: only <h1>WEB02</h1> — four times

# On LB01: HAProxy saw it die
echo "show stat" | sudo socat stdio /var/lib/haproxy/stats | cut -d, -f1,2,18 | grep web01
# Expected: web_backend,web01,DOWN

sudo tail -5 /var/log/haproxy.log
# Expected: "Server web_backend/web01 is DOWN"
```

Restart it and watch it rejoin: `sudo systemctl start httpd` on WEB01, re-run the loop — WEB01 returns to rotation within seconds.

### Test 3: VRRP Failover (the HSRP moment)

```bash
# Terminal 1, on any VM: hammer the VIP continuously
while true; do curl -s -m 2 "http://$VIP/" | grep h1 || echo "FAILED"; sleep 1; done

# Terminal 2, on LB01: kill the master
sudo systemctl stop keepalived

# Terminal 1 expected: at most 1-3 FAILED lines, then responses continue
# (VRRP advert_int 1s + 3 missed adverts ≈ 3s convergence — same as HSRP default timers)

# On LB02: VIP landed here
ip addr show "$DEV" | grep "$VIP"
# Expected: inet <VIP>/32 scope global ...

# On LB02: the VRRP story in the logs
sudo journalctl -u keepalived -n 10 --no-pager
# Expected: "Entering MASTER STATE"
```

### Test 4: Preemption

```bash
# On LB01: bring the master back
sudo systemctl start keepalived

# LB01 reclaims the VIP (priority 150 > 100)
ip addr show "$DEV" | grep "$VIP"
# Expected on LB01: VIP present again
# Expected on LB02: VIP gone, journalctl shows "Entering BACKUP STATE"
```

### Test 5: Stats Page

From any VM with a browser that can reach the flat network: `http://<LB01_IP>:8404/stats`
Or browser-free:
```bash
curl -s "http://<LB01_IP>:8404/stats;csv" | cut -d, -f1,2,18 | column -t -s,
# Expected: per-server UP/DOWN status table
```

---

## CCNA Concepts Mapped

| CCNA Topic | Where You Did It |
|-----------|------------------|
| HSRP/VRRP virtual IP | `<VIP>` floating between LB01/LB02 |
| Priority & preemption | `priority 150/100`, Test 4 re-election |
| Hello/hold timers | `advert_int 1` → ~3s failover (HSRP defaults: 3s hello, 10s hold) |
| Load balancing | HAProxy `balance roundrobin` (try `leastconn` for weighted behavior) |
| Health tracking | `option httpchk` + `fall 2 rise 1` ≈ HSRP interface tracking |
| Multicast vs unicast FHRP | VRRP normally multicasts 224.0.0.18 — you ran it unicast |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| VIP on BOTH LBs (split brain) | Peer IPs swapped? `virtual_router_id` identical? Auth pass identical? Check `journalctl -u keepalived` on both |
| VIP on NEITHER | Both in BACKUP? One must have higher priority. Also: `sudo tcpdump -i $DEV proto vrrp -c 4` — do adverts flow? |
| curl to VIP times out but VIP is up | HAProxy bound on the MASTER? `ss -tlnp \| grep :80` on the VIP holder |
| 503 from HAProxy | All backends down: `echo "show stat" \| sudo socat stdio /var/lib/haproxy/stats` |
| Backend never comes UP | `curl -m3 http://<WEB_IP>/` **from the LB** first. SELinux: `getsebool haproxy_connect_any` must be on |
| VIP responds on the wrong LB | Check which VM actually holds it: `ip addr \| grep $VIP` — curl goes wherever the IP lives |

---

## Cleanup

```bash
sudo systemctl disable --now keepalived haproxy
# If the VIP lingers after stopping keepalived:
sudo ip addr del "$VIP/32" dev "$DEV"
sudo firewall-cmd --permanent --remove-port=8404/tcp && sudo firewall-cmd --reload
sudo rm -f /etc/rsyslog.d/49-haproxy.conf && sudo systemctl restart rsyslog
# On web servers:
sudo systemctl disable --now httpd
```

---

## Next Steps

- `balance source` for **sticky sessions** (client IP hash — like cookie persistence)
- Add a **third backend** and switch to `leastconn`; watch `show stat` session counts diverge
- Track the HAProxy process in keepalived with a `vrrp_script` — VIP fails over if HAProxy itself dies, not just the box
- Terminate TLS at HAProxy with a self-signed cert (ties into your PKI lab nicely)
