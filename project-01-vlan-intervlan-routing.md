# Project 1: Inter-Subnet Routing & ACLs with a RHEL Router

**No Prism admin required.** Runs on your existing VMs, on your existing flat network.

## Goal

Turn one RHEL VM into a router serving three isolated subnets (Users / Servers / DMZ), route traffic between them, and enforce ACL-style segmentation with firewalld — the exact mental model of a Cisco router-on-a-stick with ACLs.

## Skills Covered

| CCNA | RHEL |
|------|------|
| Subnetting & broadcast domains | NetworkManager secondary IPs (`+ipv4.addresses`) |
| Router-on-a-stick / inter-VLAN routing | `sysctl net.ipv4.ip_forward` |
| Static routes | NetworkManager `+ipv4.routes` |
| Standard & extended ACLs | `firewalld` zones + rich rules with priorities |
| `show ip route` / `traceroute` verification | `ip route`, `tracepath`, `tcpdump` |

---

## How It Works Without VLANs

You can't create Nutanix VLAN networks, so instead each "VLAN" is a **separate IP subnet stacked onto the same physical wire**. Your router VM holds the gateway address for all three subnets on its single vNIC; hosts point at the router for foreign subnets. Traffic hairpins through the router — which is **identical routing behavior to inter-VLAN routing**. You lose L2 broadcast isolation (a switch-side skill), and keep every routing/ACL skill.

```
              Existing flat network (one wire)
    ┌───────────────────────────────────────────────────┐
    │                                                   │
    │   "Users" 10.230.10.0/24   (secondary IPs)       │
    │   "Servers" 10.230.20.0/24                       │
    │   "DMZ"    10.230.30.0/24                        │
    │                                                   │
    │   ┌────────┐    ┌────────┐    ┌────────┐          │
    │   │ USER01 │    │ SVR01  │    │ DMZ01  │          │
    │   │ .10.10 │    │ .20.10 │    │ .30.10 │          │
    │   └───┬────┘    └───┬────┘    └───┬────┘          │
    │       │             │             │               │
    │       └─────────────┼─────────────┘               │
    │                     │ (all hairpin via R1)        │
    │               ┌─────┴──────┐                      │
    │               │     R1     │                      │
    │               │ .10.1      │                      │
    │               │ .20.1      │  (all on one vNIC)   │
    │               │ .30.1      │                      │
    │               └────────────┘                      │
    └───────────────────────────────────────────────────┘
```

---

## Prerequisites

- [ ] 4 existing RHEL VMs (R1, USER01, SVR01, DMZ01) — or 2 VMs minimum (R1 + one host; add others later)
- [ ] sudo on all of them
- [ ] Your flat network does **not** already use 10.230.10.0/24, 10.230.20.0/24, or 10.230.30.0/24 (these guides deliberately use the uncommon `10.230.x` block to avoid the usual 172.x / 192.168.x corporate ranges). If `10.230.x` is somehow also taken, pick another /24 block (e.g. 10.231.x) and shift everywhere below
- [ ] Firewalld running: `systemctl status firewalld`

---

## Step 1: Discover Your Environment (every VM)

```bash
DEV=$(ip route show default | awk '{print $5; exit}')
echo "$DEV"
# Expected: your NIC name, e.g. ens3 / eth0 / enp1s0

nmcli -f NAME,DEVICE con show --active
# Note the NAME on your device — this is <flat-con>

FLAT_IP=$(ip -4 -o addr show dev "$DEV" scope global | awk 'NR==1{print $4}')
FLAT_GW=$(ip route show default | awk '{print $3; exit}')
EXISTING_DNS=$(nmcli -f IP4.DNS dev show "$DEV" | awk '{print $2}' | paste -sd,)
echo "$FLAT_IP  via  $FLAT_GW  dns=$EXISTING_DNS"
# Expected: your current IP/CIDR, gateway, DNS — SAVE THESE, Step 2 reapplies them
```

---

## Step 2: Configure the Router (R1)

### 2.1 Stack the Three Subnet Gateways on the Existing NIC

We convert the connection from DHCP to manual, keeping the flat IP (management) and adding the three lab gateways:

```bash
sudo nmcli con mod "<flat-con>" ipv4.method manual \
    ipv4.addresses "$FLAT_IP,10.230.10.1/24,10.230.20.1/24,10.230.30.1/24" \
    ipv4.gateway "$FLAT_GW" \
    ipv4.dns "$EXISTING_DNS"

sudo nmcli con up "<flat-con>"
```

**Verify:**
```bash
ip -4 addr show "$DEV" | grep inet
# Expected: your flat IP PLUS:
# inet 10.230.10.1/24 ...
# inet 10.230.20.1/24 ...
# inet 10.230.30.1/24 ...

ssh still works on the flat IP? (open a second session to be sure before continuing)
```

### 2.2 Enable Forwarding, Disable Redirects

Redirects would tell hosts to bypass the router — killing your ACL demo. Disable both sending (router) and accepting (hosts, Step 3).

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.conf.all.send_redirects=0
sudo sysctl -w net.ipv4.conf.default.send_redirects=0

cat << 'EOF' | sudo tee /etc/sysctl.d/90-router.conf
net.ipv4.ip_forward=1
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0
EOF

sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1
```

### 2.3 The "ACL" — firewalld Policy

Cisco equivalent being built:

```
! permit DMZ -> Servers tcp/80 only
! deny    DMZ -> everything else
! permit  all other traffic
```

```bash
sudo firewall-cmd --permanent --new-zone=lab
sudo firewall-cmd --permanent --zone=lab --add-source=10.230.10.0/24
sudo firewall-cmd --permanent --zone=lab --add-source=10.230.20.0/24
sudo firewall-cmd --permanent --zone=lab --add-source=10.230.30.0/24

# ACL entries — priorities make evaluation order deterministic (lowest first)
# deny DMZ -> Users (extended ACL: deny ip 10.230.30.0 0.0.0.255 10.230.10.0 0.0.0.255)
sudo firewall-cmd --permanent --zone=lab --add-rich-rule='rule priority=-100 family=ipv4 source address=10.230.30.0/24 destination address=10.230.10.0/24 reject'

# permit DMZ -> Servers tcp/80 (permit tcp 10.230.30.0 0.0.0.255 10.230.20.0 0.0.0.255 eq 80)
sudo firewall-cmd --permanent --zone=lab --add-rich-rule='rule priority=-99 family=ipv4 source address=10.230.30.0/24 destination address=10.230.20.0/24 port port=80 protocol=tcp accept'

# deny remaining DMZ -> Servers
sudo firewall-cmd --permanent --zone=lab --add-rich-rule='rule priority=-98 family=ipv4 source address=10.230.30.0/24 destination address=10.230.20.0/24 reject'

# Everything else: permit (target ACCEPT = "permit ip any any" at end of ACL)
sudo firewall-cmd --permanent --zone=lab --set-target=ACCEPT

sudo firewall-cmd --reload
```

**Verify:**
```bash
sudo firewall-cmd --zone=lab --list-all
# Expected: lab zone, 3 sources, 3 rich rules in priority order, target: ACCEPT
```

---

## Step 3: Configure the Hosts

Run on **each host**, with its own lab IP. Each host keeps its flat IP for management and gets a lab IP + static routes to the *other* lab subnets via R1.

### USER01 (10.230.10.10)

```bash
# After running Step 1 discovery on USER01:
sudo nmcli con mod "<flat-con>" ipv4.method manual \
    ipv4.addresses "$FLAT_IP,10.230.10.10/24" \
    ipv4.gateway "$FLAT_GW" \
    ipv4.dns "$EXISTING_DNS" \
    +ipv4.routes "10.230.20.0/24 10.230.10.1" \
    +ipv4.routes "10.230.30.0/24 10.230.10.1"

sudo nmcli con up "<flat-con>"

# Don't accept ICMP redirects (keeps traffic pinned to R1)
echo "net.ipv4.conf.all.accept_redirects=0" | sudo tee /etc/sysctl.d/91-no-redirects.conf
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0
```

### SVR01 (10.230.20.10)

```bash
sudo nmcli con mod "<flat-con>" ipv4.method manual \
    ipv4.addresses "$FLAT_IP,10.230.20.10/24" \
    ipv4.gateway "$FLAT_GW" \
    ipv4.dns "$EXISTING_DNS" \
    +ipv4.routes "10.230.10.0/24 10.230.20.1" \
    +ipv4.routes "10.230.30.0/24 10.230.20.1"

sudo nmcli con up "<flat-con>"
echo "net.ipv4.conf.all.accept_redirects=0" | sudo tee /etc/sysctl.d/91-no-redirects.conf
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0

# Optional: web server so the DMZ->Servers:80 permit has something to hit
sudo dnf install -y httpd && sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http && sudo firewall-cmd --reload
```

### DMZ01 (10.230.30.10)

```bash
sudo nmcli con mod "<flat-con>" ipv4.method manual \
    ipv4.addresses "$FLAT_IP,10.230.30.10/24" \
    ipv4.gateway "$FLAT_GW" \
    ipv4.dns "$EXISTING_DNS" \
    +ipv4.routes "10.230.10.0/24 10.230.30.1" \
    +ipv4.routes "10.230.20.0/24 10.230.30.1"

sudo nmcli con up "<flat-con>"
echo "net.ipv4.conf.all.accept_redirects=0" | sudo tee /etc/sysctl.d/91-no-redirects.conf
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0
```

---

## Step 4: Validation Tests

> ⚠️ Use **lab IPs only** in these tests. Pinging a VM's flat IP bypasses the router and proves nothing.

### Test 1: Host Reaches Its Gateway

```bash
# On USER01
ping -c 3 10.230.10.1
# Expected: 3 replies from 10.230.10.1
```

### Test 2: Inter-Subnet Routing (Permitted)

```bash
# On USER01
ping -c 3 10.230.20.10
# Expected: 3 replies

tracepath 10.230.20.10
# Expected: 1: 10.230.10.1   2: 10.230.20.10
# (hairpin through R1 — the inter-VLAN behavior)
```

### Test 3: ACL — DMZ Cannot Reach Users

```bash
# On DMZ01
ping -c 3 10.230.10.10
# Expected: "Destination Port Unreachable" / "Packet filtered" — 100% loss
```

### Test 4: ACL — DMZ Reaches Servers on TCP/80 Only

```bash
# On DMZ01
ping -c 3 10.230.20.10
# Expected: filtered/unreachable — ICMP is NOT permitted DMZ->Servers

curl -s -m 5 http://10.230.20.10/ | head -3
# Expected: HTML (or "Failed to connect" only if you skipped httpd on SVR01)
```

### Test 5: Watch the Router Work

```bash
# On R1, while DMZ01 retries Test 3
sudo tcpdump -i "$DEV" -n host 10.230.30.10 and icmp -c 6
# Expected: echo requests arriving AND R1's icmp admin-prohibited replies going back
```

### Test 6: Routing Table (the `show ip route` moment)

```bash
# On R1
ip route show
# Expected: connected routes for all three 10.230.x.0/24 subnets + your flat routes
```

---

## Optional Extension: Second Router + Static Routes

Have a 5th VM? Build the classic two-router topology on the same wire:

```bash
# On R2: stack gateways as .254 plus a transit address
sudo nmcli con mod "<flat-con>" ipv4.method manual \
    ipv4.addresses "$FLAT_IP,10.230.10.254/24,10.230.20.254/24,10.230.99.2/30" \
    ipv4.gateway "$FLAT_GW" ipv4.dns "$EXISTING_DNS"
sudo nmcli con up "<flat-con>"
sudo sysctl -w net.ipv4.ip_forward=1

# On R1: add transit address + route "half the world" via R2
sudo nmcli con mod "<flat-con>" +ipv4.addresses "10.230.99.1/30" \
    +ipv4.routes "10.230.30.0/24 10.230.99.2"
sudo nmcli con up "<flat-con>"

ip route show | grep 10.230.99
# Expected on R1: 10.230.99.0/30 dev ... + 10.230.30.0/24 via 10.230.99.2
```

---

## CCNA Concepts Mapped

| CCNA Topic | Where You Did It |
|-----------|------------------|
| Subnets / broadcast domains | 3 secondary subnets on one wire |
| Router-on-a-stick | R1 holds .1 of all 3 subnets on one vNIC |
| Static routes | `+ipv4.routes` on hosts and routers |
| Extended ACL | firewalld rich rules with priorities (-100 deny, -99 permit tcp/80, -98 deny) |
| Implicit deny | Demonstrated by flipping `--set-target` (try REJECT and watch everything die) |
| Verification commands | `ip route`, `tracepath`, `tcpdump` ≈ `show ip route`, `traceroute`, `debug ip packet` |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Lost SSH after `con mod` | You changed the flat IP. Console in, `nmcli con show "<flat-con>" \| grep ipv4` |
| Host can't ping gateway | `ip addr show` on host — lab IP present? On R1 — all three .1 addresses present? |
| Ping works but tracepath shows direct path | Redirects accepted. `sysctl net.ipv4.conf.all.accept_redirects` on host must be 0; `send_redirects` on R1 must be 0 |
| DMZ can ping Users | Rich rules missing: `sudo firewall-cmd --zone=lab --list-all`. Also confirm sources are in the zone |
| DMZ can't curl :80 either | httpd running on SVR01? `curl -m5 http://10.230.20.10/` **from SVR01 itself** first |
| Nothing forwards at all | `sysctl net.ipv4.ip_forward` on R1 must be 1 |

---

## Cleanup

```bash
# On each host: remove the lab IP and routes
sudo nmcli con mod "<flat-con>" -ipv4.addresses "10.230.10.10/24" \
    -ipv4.routes "10.230.20.0/24 10.230.10.1" -ipv4.routes "10.230.30.0/24 10.230.10.1"
sudo nmcli con up "<flat-con>"
sudo rm -f /etc/sysctl.d/91-no-redirects.conf

# On R1
sudo nmcli con mod "<flat-con>" -ipv4.addresses "10.230.10.1/24" \
    -ipv4.addresses "10.230.20.1/24" -ipv4.addresses "10.230.30.1/24"
sudo nmcli con up "<flat-con>"
sudo firewall-cmd --permanent --delete-zone=lab && sudo firewall-cmd --reload
sudo rm -f /etc/sysctl.d/90-router.conf
```

---

## Next Steps

- Flip the zone target to `REJECT` and write **permit-only** rich rules (true implicit-deny ACL behavior)
- Install `frr` and replace static routes with **OSPF** between R1 and R2
- Carry this topology into Project 3: put one subnet behind a WireGuard tunnel
