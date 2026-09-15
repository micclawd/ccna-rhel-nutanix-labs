# Project 1: VLAN-Isolated Multi-Site Network with RHEL Routers

## Goal

Build a 3-VLAN network in Nutanix with two RHEL VMs acting as routers, enabling inter-VLAN routing while enforcing ACL-based segmentation.

## Skills Covered

| CCNA | RHEL |
|------|------|
| VLANs & 802.1Q trunking | NetworkManager (nmcli/nmtui) |
| Inter-VLAN routing | firewall-cmd (zones, rich rules) |
| Static routing | sysctl (net.ipv4.ip_forward) |
| Standard & Extended ACLs | Network interface configuration |

---

## Topology

```
                    Nutanix AHV Cluster
    ┌─────────────────────────────────────────────────┐
    │                                                 │
    │   VLAN10 (Users)    VLAN20 (Servers)  VLAN30 (DMZ) │
    │   192.168.10.0/24   192.168.20.0/24   192.168.30.0/24 │
    │        │                 │                │      │
    │        │                 │                │      │
    │   ┌────┴────┐       ┌────┴────┐      ┌────┴────┐ │
    │   │  USER01 │       │  SVR01  │      │  DMZ01  │ │
    │   │ .10.10  │       │ .20.10  │      │ .30.10  │ │
    │   └────┬────┘       └────┬────┘      └────┬────┘ │
    │        │                 │                │      │
    │        └─────────────────┼────────────────┘      │
    │                          │                       │
    │                    ┌─────┴─────┐                 │
    │                    │    R1     │                 │
    │                    │  Router   │                 │
    │                    │ .10.1     │                 │
    │                    │ .20.1     │                 │
    │                    │ .30.1     │                 │
    │                    └─────┬─────┘                 │
    │                          │                       │
    │                    ┌─────┴─────┐                 │
    │                    │    R2     │                 │
    │                    │  Router   │                 │
    │                    │ .10.254   │                 │
    │                    │ .20.254   │                 │
    │                    │ .30.254   │                 │
    │                    └───────────┘                 │
    │                                                 │
    └─────────────────────────────────────────────────┘
```

**Note:** R1 and R2 are connected via a **transit network** (VLAN99, 10.0.99.0/30) for static route exchange. In a real CCNA lab this would be a serial link or Ethernet between routers.

---

## Prerequisites

- [ ] Nutanix Prism access
- [ ] RHEL 9.x ISO uploaded to Prism > Images
- [ ] 5 VMs created from RHEL ISO (2 routers, 3 end hosts)
- [ ] Each VM has console access via Prism

---

## Step 1: Create Nutanix Networks

In **Prism > Network > Network Config**, create these networks:

| Network Name | VLAN ID | Subnet | Gateway (Nutanix) |
|-------------|---------|--------|-------------------|
| VLAN10-Users | 10 | 192.168.10.0/24 | 192.168.10.1 |
| VLAN20-Servers | 20 | 192.168.20.0/24 | 192.168.20.1 |
| VLAN30-DMZ | 30 | 192.168.30.0/24 | 192.168.30.1 |
| VLAN99-Transit | 99 | 10.0.99.0/30 | 10.0.99.1 |

> **Important:** In Nutanix AHV, the "Gateway" you configure is the IP that Nutanix itself responds on. For our lab, we'll override this with static IPs on the RHEL routers.

---

## Step 2: Configure RHEL Router 1 (R1)

### 2.1 Assign IP Addresses

R1 has **4 vNICs**: one on each VLAN (10, 20, 30) and one on VLAN99 (transit to R2).

```bash
# Check interface names
ip link show

# Expected output: ens3, ens4, ens5, ens6 (or similar)

# Configure VLAN10 interface
sudo nmcli con add type ethernet ifname ens3 con-name vlan10 \
    ipv4.method manual ipv4.addresses 192.168.10.1/24 \
    ipv4.gateway "" ipv4.dns ""

# Configure VLAN20 interface
sudo nmcli con add type ethernet ifname ens4 con-name vlan20 \
    ipv4.method manual ipv4.addresses 192.168.20.1/24 \
    ipv4.gateway "" ipv4.dns ""

# Configure VLAN30 interface
sudo nmcli con add type ethernet ifname ens5 con-name vlan30 \
    ipv4.method manual ipv4.addresses 192.168.30.1/24 \
    ipv4.gateway "" ipv4.dns ""

# Configure VLAN99 transit interface
sudo nmcli con add type ethernet ifname ens6 con-name vlan99 \
    ipv4.method manual ipv4.addresses 10.0.99.1/30 \
    ipv4.gateway "" ipv4.dns ""

# Bring all up
sudo nmcli con up vlan10
sudo nmcli con up vlan20
sudo nmcli con up vlan30
sudo nmcli con up vlan99
```

**Verify:**
```bash
ip -4 addr show | grep -E "inet (192|10)"
# Expected:
# inet 192.168.10.1/24 ...
# inet 192.168.20.1/24 ...
# inet 192.168.30.1/24 ...
# inet 10.0.99.1/30 ...
```

### 2.2 Enable IP Forwarding

```bash
# Enable immediately
sudo sysctl -w net.ipv4.ip_forward=1

# Make persistent
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/90-ipforward.conf

# Verify
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1
```

### 2.3 Add Static Routes to R2

```bash
# Route to VLAN10 via R2
sudo nmcli con mod vlan99 +ipv4.routes "192.168.10.0/24 10.0.99.2"

# Route to VLAN20 via R2
sudo nmcli con mod vlan99 +ipv4.routes "192.168.20.0/24 10.0.99.2"

# Route to VLAN30 via R2
sudo nmcli con mod vlan99 +ipv4.routes "192.168.30.0/24 10.0.99.2"

# Apply
sudo nmcli con down vlan99 && sudo nmcli con up vlan99

# Verify
ip route show | grep 192.168
# Expected:
# 192.168.10.0/24 via 10.0.99.2 dev ens6
# 192.168.20.0/24 via 10.0.99.2 dev ens6
# 192.168.30.0/24 via 10.0.99.2 dev ens6
```

### 2.4 Configure Firewall (ACL Equivalent)

On RHEL, `firewalld` rich rules act like Cisco extended ACLs.

```bash
# Allow forwarding between VLAN10 and VLAN20 (permit)
sudo firewall-cmd --permanent --new-zone=internal-users
sudo firewall-cmd --permanent --zone=internal-users --add-source=192.168.10.0/24
sudo firewall-cmd --permanent --zone=internal-users --add-forward

# Block DMZ (VLAN30) from reaching Users (VLAN10) — like an ACL
sudo firewall-cmd --permanent --new-zone=dmz-restricted
sudo firewall-cmd --permanent --zone=dmz-restricted --add-source=192.168.30.0/24
sudo firewall-cmd --permanent --zone=dmz-restricted --add-rich-rule='rule family=ipv4 destination address=192.168.10.0/24 reject'

# Allow DMZ to reach Servers only on port 80 (like a permit tcp any host eq 80)
sudo firewall-cmd --permanent --zone=dmz-restricted --add-rich-rule='rule family=ipv4 destination address=192.168.20.0/24 port port=80 protocol=tcp accept'

# Assign interfaces to zones
sudo firewall-cmd --permanent --zone=internal-users --change-interface=ens3
sudo firewall-cmd --permanent --zone=dmz-restricted --change-interface=ens5

# Enable masquerade for outbound (like NAT overload)
sudo firewall-cmd --permanent --zone=public --add-masquerade

sudo firewall-cmd --reload
```

---

## Step 3: Configure RHEL Router 2 (R2)

R2 mirrors R1 but with .254 addresses and routes pointing back to R1.

```bash
# VLAN10 interface
sudo nmcli con add type ethernet ifname ens3 con-name vlan10 \
    ipv4.method manual ipv4.addresses 192.168.10.254/24

# VLAN20 interface
sudo nmcli con add type ethernet ifname ens4 con-name vlan20 \
    ipv4.method manual ipv4.addresses 192.168.20.254/24

# VLAN30 interface
sudo nmcli con add type ethernet ifname ens5 con-name vlan30 \
    ipv4.method manual ipv4.addresses 192.168.30.254/24

# VLAN99 transit
sudo nmcli con add type ethernet ifname ens6 con-name vlan99 \
    ipv4.method manual ipv4.addresses 10.0.99.2/30

sudo nmcli con up vlan10 && sudo nmcli con up vlan20 && \
sudo nmcli con up vlan30 && sudo nmcli con up vlan99

# Enable forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/90-ipforward.conf

# Routes back to R1
sudo nmcli con mod vlan99 +ipv4.routes "192.168.10.0/24 10.0.99.1"
sudo nmcli con mod vlan99 +ipv4.routes "192.168.20.0/24 10.0.99.1"
sudo nmcli con mod vlan99 +ipv4.routes "192.168.30.0/24 10.0.99.1"
sudo nmcli con down vlan99 && sudo nmcli con up vlan99
```

---

## Step 4: Configure End Hosts

### USER01 (VLAN10)

```bash
sudo nmcli con add type ethernet ifname ens3 con-name users \
    ipv4.method manual ipv4.addresses 192.168.10.10/24 \
    ipv4.gateway 192.168.10.1 ipv4.dns "8.8.8.8"

sudo nmcli con up users
```

### SVR01 (VLAN20)

```bash
sudo nmcli con add type ethernet ifname ens3 con-name servers \
    ipv4.method manual ipv4.addresses 192.168.20.10/24 \
    ipv4.gateway 192.168.20.1 ipv4.dns "8.8.8.8"

sudo nmcli con up servers
```

### DMZ01 (VLAN30)

```bash
sudo nmcli con add type ethernet ifname ens3 con-name dmz \
    ipv4.method manual ipv4.addresses 192.168.30.10/24 \
    ipv4.gateway 192.168.30.1 ipv4.dns "8.8.8.8"

sudo nmcli con up dmz
```

---

## Step 5: Validation Tests

### Test 1: Intra-VLAN Communication

```bash
# From USER01, ping its gateway (R1)
ping -c 3 192.168.10.1
# Expected: 3 replies

# From USER01, ping R2 on same VLAN
ping -c 3 192.168.10.254
# Expected: 3 replies (both routers respond on VLAN10)
```

### Test 2: Inter-VLAN Routing (Permitted)

```bash
# From USER01 (VLAN10), ping SVR01 (VLAN20)
ping -c 3 192.168.20.10
# Expected: 3 replies — traffic routes via R1

# Traceroute to see path
tracepath 192.168.20.10
# Expected: 192.168.10.1 → 192.168.20.10
```

### Test 3: DMZ Restriction (ACL Block)

```bash
# From DMZ01, try to ping USER01 — should FAIL
ping -c 3 192.168.10.10
# Expected: 100% packet loss (or "Destination Port Unreachable")

# From DMZ01, ping SVR01 on ICMP — should FAIL
ping -c 3 192.168.20.10
# Expected: 100% packet loss

# From DMZ01, curl SVR01 on port 80 — should SUCCEED (if httpd installed)
curl -s http://192.168.20.10/ | head -5
# Expected: HTML response or connection refused (if no web server), but NOT timeout
```

### Test 4: Static Route Verification

```bash
# On R1, check routing table
ip route show
# Expected: routes for all 3 VLANs, some direct, some via 10.0.99.2

# On R2, same
ip route show
# Expected: routes for all 3 VLANs, some direct, some via 10.0.99.1
```

---

## CCNA Concepts Mapped

| CCNA Topic | How It's Demonstrated |
|-----------|----------------------|
| VLANs | 3 isolated broadcast domains in Nutanix |
| 802.1Q | Nutanix AHV handles tagging; VMs see untagged traffic |
| Inter-VLAN routing | RHEL router with multiple VLAN interfaces |
| Static routes | `ipv4.routes` in NetworkManager |
| Standard ACL | `firewall-cmd --zone=... --add-source` |
| Extended ACL | `--add-rich-rule` with destination + port |
| Routing table | `ip route show` |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No inter-VLAN ping | `sysctl net.ipv4.ip_forward` on router |
| Wrong interface IPs | `nmcli con show <name>` — verify `ipv4.addresses` |
| DMZ can ping Users | Check firewalld zone assignment: `firewall-cmd --get-active-zones` |
| Routes not loading | `nmcli con down/up` after modifying routes |
| Nutanix network not passing traffic | Verify VLAN ID matches between Prism and VM config |

---

## Cleanup

```bash
# On routers: remove all connections
sudo nmcli con delete vlan10 vlan20 vlan30 vlan99

# On hosts: remove connection
sudo nmcli con delete users servers dmz

# In Prism: delete networks if desired
```

---

## Next Steps

- Add **OSPF** or **EIGRP** between R1 and R2 using `frr` (FRRouting) package
- Configure **DHCP relay** on R1 to forward to a central DHCP server
- Add **NAT overload** (PAT) on R2 for outbound internet simulation
