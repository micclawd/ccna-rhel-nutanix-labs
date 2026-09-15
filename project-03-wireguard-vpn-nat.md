# Project 3: Site-to-Site VPN with WireGuard + NAT

**No Prism admin required.** Runs on 2 existing VMs — WireGuard is a pure overlay, so your existing flat network IS the "WAN." Zero network changes needed anywhere.

## Goal

Tunnel a private site subnet between two RHEL gateways over the flat network, with NAT — the exact mental model of a Cisco site-to-site IPsec VPN with PAT.

## Skills Covered

| CCNA | RHEL |
|------|------|
| Site-to-site VPN tunnel establishment | WireGuard (`wg-quick`, keypairs) |
| NAT/PAT (inside/outside) | `firewalld` masquerade, `conntrack` |
| Tunnel vs transport mode concepts | WireGuard's UDP encapsulation |
| `show crypto` verification | `wg show`, handshake/transfer counters |
| Encrypted payload proof | `tcpdump` on the underlay |

---

## Topology

```
            Existing flat network ("the WAN" — no changes needed)
    ┌──────────────────────────────────────────────────┐
    │                                                  │
    │   GW-A (flat IP)  ══════ UDP 51820 ══════  GW-B (flat IP) │
    │        │          encrypted tunnel         │     │
    │        │                                   │     │
    │   ┌────┴─────┐                       ┌─────┴───┐ │
    │   │ dummy0   │                       │ dummy0  │ │
    │   │ Site-A   │                       │ Site-B  │ │
    │   │10.230.  │  ← routed via wg0 →   │10.230. │ │
    │   │10.1/24   │                       │20.1/24  │ │
    │   └──────────┘                       └─────────┘ │
    │                                                  │
    │   wg0: 10.230.100.1/30                   wg0: 10.230.100.2/30 │
    │                                                  │
    └──────────────────────────────────────────────────┘
```

- **Flat IPs** = underlay endpoints (already exist, untouched)
- **wg0 / 10.230.100.0/30** = the tunnel itself
- **dummy0 / 10.230.x.0/24** = private "site LANs" that only exist on each gateway — reachable from the other side *only* through the tunnel

---

## Prerequisites

- [ ] 2 existing RHEL VMs (GW-A, GW-B), sudo on both
- [ ] `wireguard-tools` and `kernel` modules available

**Air-gapped install** — wireguard-tools is in EPEL (not the base ISO). Options in order:

```bash
# Option A: already installed?
which wg && modinfo wireguard && echo OK

# Option B: pre-staged RPMs (copy wireguard-tools-*.rpm + kmod deps to the VMs)
sudo rpm -ivh wireguard-tools-*.rpm

# Option C: local repo from a staged EPEL mirror directory
sudo tee /etc/yum.repos.d/local-epel.repo << 'EOF'
[local-epel]
name=Local EPEL
baseurl=file:///opt/epel
enabled=1
gpgcheck=0
EOF
sudo dnf install -y wireguard-tools

# Verify the kernel module loads (RHEL 8.6+/9 has it in-tree)
sudo modprobe wireguard && lsmod | grep wireguard
# Expected: wireguard <size> 0
```

---

## Step 0: Discovery (both VMs)

```bash
FLAT_IP=$(ip -4 -o addr show scope global | awk 'NR==1{split($4,a,"/");print a[1]}')
echo "$FLAT_IP"
# WRITE DOWN both flat IPs — they become the tunnel endpoints

sudo hostnamectl set-hostname gw-a.lab.local   # GW-A
sudo hostnamectl set-hostname gw-b.lab.local   # GW-B
```

---

## Step 1: Gateway A

### 1.1 Keys

```bash
sudo mkdir -p /etc/wireguard && sudo chmod 700 /etc/wireguard
cd /etc/wireguard

wg genkey | sudo tee private.key | wg pubkey | sudo tee public.key
sudo chmod 600 private.key

sudo cat public.key
# Expected: 44-char base64 string ending in '=' — COPY IT, GW-B needs it
```

### 1.2 Tunnel Config

```bash
sudo tee /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = $(sudo cat /etc/wireguard/private.key)
Address = 10.230.100.1/30
ListenPort = 51820

[Peer]
# GW-B's public key — PASTE after configuring GW-B (Step 2.1)
PublicKey = PASTE_GWB_PUBLIC_KEY_HERE
AllowedIPs = 10.230.100.0/30, 10.230.20.0/24
Endpoint = <GWB_FLAT_IP>:51820
PersistentKeepalive = 25
EOF

sudo chmod 600 /etc/wireguard/wg0.conf
```

> Replace `<GWB_FLAT_IP>` with GW-B's actual flat IP now; the `PublicKey` placeholder gets fixed in Step 3.

### 1.3 Site Subnet on a Dummy Interface

```bash
sudo nmcli con add type dummy ifname dummy0 \
    ipv4.method manual ipv4.addresses 10.230.10.1/24
sudo nmcli con up dummy0

ip addr show dummy0
# Expected: inet 10.230.10.1/24
```

### 1.4 Forwarding + NAT + Firewall

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/90-ipforward.conf
sudo sysctl -w net.ipv4.ip_forward=1

sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --permanent --zone=public --add-port=51820/udp
sudo firewall-cmd --reload

sudo firewall-cmd --list-all
# Expected: masquerade: yes, ports: 51820/udp
```

### 1.5 Bring Up the Tunnel

```bash
sudo systemctl enable --now wg-quick@wg0
sudo wg show
# Expected: interface wg0, listening port 51820, peer with endpoint <GWB_FLAT_IP>:51820
# (handshake will be 0 until GW-B is up and keys are exchanged)
```

---

## Step 2: Gateway B (mirror)

```bash
# 2.1 Keys
sudo mkdir -p /etc/wireguard && sudo chmod 700 /etc/wireguard
cd /etc/wireguard
wg genkey | sudo tee private.key | wg pubkey | sudo tee public.key
sudo chmod 600 private.key
sudo cat public.key
# COPY THIS — it goes into GW-A's config in Step 3

# 2.2 Config
sudo tee /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = $(sudo cat /etc/wireguard/private.key)
Address = 10.230.100.2/30
ListenPort = 51820

[Peer]
PublicKey = <GWA_PUBLIC_KEY_FROM_STEP_1.1>
AllowedIPs = 10.230.100.0/30, 10.230.10.0/24
Endpoint = <GWA_FLAT_IP>:51820
PersistentKeepalive = 25
EOF
sudo chmod 600 /etc/wireguard/wg0.conf

# 2.3 Site subnet
sudo nmcli con add type dummy ifname dummy0 \
    ipv4.method manual ipv4.addresses 10.230.20.1/24
sudo nmcli con up dummy0

# 2.4 Forwarding + NAT + firewall
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/90-ipforward.conf
sudo sysctl -w net.ipv4.ip_forward=1
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --permanent --zone=public --add-port=51820/udp
sudo firewall-cmd --reload

# 2.5 Up
sudo systemctl enable --now wg-quick@wg0
```

---

## Step 3: Finish the Key Exchange

```bash
# On GW-A: insert GW-B's real public key
sudo sed -i "s|PASTE_GWB_PUBLIC_KEY_HERE|<GWB_PUBLIC_KEY>|" /etc/wireguard/wg0.conf
sudo systemctl restart wg-quick@wg0

# On both: verify the handshake
sudo wg show
```

**Expected (the `show crypto ipsec sa` moment):**
```
interface: wg0
  public key: <this side's key>
  listening port: 51820

peer: <other side's key>
  endpoint: <other side's flat IP>:51820
  allowed ips: 10.230.100.0/30, 10.230.x.0/24
  latest handshake: 12 seconds ago        ← tunnel is UP
  transfer: 148 B received, 180 B sent
```

---

## Step 4: Validation Tests

### Test 1: Tunnel Reachability

```bash
# On GW-A
ping -c 3 10.230.100.2
# Expected: 3 replies — this ONLY works through wg0
```

### Test 2: Cross-Site Reachability

```bash
# On GW-A: reach Site-B's private subnet through the tunnel
ping -c 3 10.230.20.1
# Expected: 3 replies

tracepath 10.230.20.1
# Expected: 1: 10.230.100.2   2: 10.230.20.1
# (NOT via the flat default gateway — that's the routing win)
```

### Test 3: Proof of Encryption

```bash
# On GW-A, capture the underlay while pinging 10.230.20.1 from another shell
sudo tcpdump -i <flat-dev> -n udp port 51820 -c 4
# Expected: UDP packets between the two flat IPs — opaque encrypted payload.
# You will NOT see the inner ICMP. That's the CCNA "tunnel encapsulation" point.
```

### Test 4: NAT/PAT

```bash
# On GW-A: ping a third flat-network VM, sourced from the site subnet
ping -I 10.230.10.1 -c 4 <SOME_OTHER_FLAT_VM_IP>
# Expected: replies — the target sees GW-A's flat IP, not 10.230.10.1

# NAT counters climbing:
sudo iptables -t nat -L POSTROUTING -v -n
# Expected: pkts/bytes incrementing on the MASQUERADE rule

# Live translations:
sudo cat /proc/net/nf_conntrack | grep 10.230.10.1 | head -3
# Expected: conntrack entries showing 10.230.10.1 → GW-A flat IP rewrite
```

### Test 5: Split Tunneling Behavior

```bash
# On GW-A: flat-network traffic does NOT enter the tunnel
ping -c 2 <GWB_FLAT_IP>
# Then check: AllowedIPs decides what enters wg0
sudo wg show | grep -A2 peer
# Expected: transfer counters from Test 5's flat ping did NOT increment tunnel traffic
```

---

## Optional: Add a Real Client Behind a Gateway

Have a 3rd VM? Make it a Site-A host and route through GW-A:

```bash
# On the 3rd VM (CLIENT-A)
DEV=$(ip route show default | awk '{print $5; exit}')
FLAT_IP=$(ip -4 -o addr show dev "$DEV" scope global | awk 'NR==1{print $4}')
nmcli -f NAME,DEVICE con show --active

sudo nmcli con mod "<flat-con>" ipv4.method manual \
    ipv4.addresses "$FLAT_IP,10.230.10.10/24" \
    ipv4.gateway "$(ip route show default | awk '{print $3; exit}')" \
    +ipv4.routes "10.230.20.0/24 10.230.10.1"
sudo nmcli con up "<flat-con>"

echo "net.ipv4.conf.all.accept_redirects=0" | sudo tee /etc/sysctl.d/91-no-redirects.conf
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0

# Now the real test — CLIENT-A reaches Site-B across the tunnel:
ping -c 3 10.230.20.1
# Expected: replies. Path: CLIENT-A → GW-A → wg0 → GW-B → dummy0
```

---

## CCNA Concepts Mapped

| CCNA Topic | Where You Did It |
|-----------|------------------|
| Site-to-site VPN | wg0 tunnel between two gateways over the underlay |
| Tunnel endpoints | `Endpoint` = peer's flat IP (like crypto map peer) |
| Interesting traffic | `AllowedIPs` = the crypto ACL — only matching traffic enters the tunnel |
| NAT inside→outside | Masquerade on the flat zone; conntrack shows the rewrite |
| VPN verification | `wg show` handshake/transfer ≈ `show crypto ipsec sa` |
| Encryption proof | tcpdump on underlay shows only UDP 51820, no inner traffic |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No handshake | Keys swapped correctly? `sudo wg show` on both sides — each side's peer key must be the OTHER side's public key. Endpoint IPs reachable? `ping <peer-flat-ip>` |
| Handshake OK, can't ping site subnet | `AllowedIPs` on the *sending* side must include the remote site subnet; `ip_forward=1` on both |
| Site subnet ping dies | dummy0 up? `ip addr show dummy0`. Route exists? `ip route \| grep 10.230` — should show via wg0 |
| wg-quick fails at boot | `journalctl -u wg-quick@wg0 -n 30` — usually a config typo |
| NAT counter stays 0 | Traffic must EXIT the flat interface from a 10.230.x source; check `firewall-cmd --get-active-zones` |

---

## Cleanup

```bash
# Both gateways
sudo systemctl disable --now wg-quick@wg0
sudo nmcli con delete dummy0
sudo firewall-cmd --permanent --zone=public --remove-masquerade
sudo firewall-cmd --permanent --zone=public --remove-port=51820/udp
sudo firewall-cmd --reload
sudo rm -f /etc/wireguard/wg0.conf /etc/wireguard/*.key /etc/sysctl.d/90-ipforward.conf
```

---

## Next Steps

- **Hub-and-spoke:** add GW-C with its own site subnet; GW-A becomes the hub
- **Compare with OpenVPN** (`openvpn` package) — same topology, SSL/TLS instead of WireGuard
- **Dynamic routing over the tunnel:** install `frr`, run OSPF across 10.230.100.0/30
