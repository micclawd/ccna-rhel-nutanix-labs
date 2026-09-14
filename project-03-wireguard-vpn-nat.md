# Project 3: Site-to-Site VPN with WireGuard + NAT

## Goal

Build a site-to-site VPN tunnel between two isolated networks using WireGuard on RHEL gateways, with NAT for outbound traffic.

## Skills Covered

| CCNA | RHEL |
|------|------|
| Site-to-site VPN concepts | WireGuard installation & configuration |
| NAT/PAT (overload) | `firewalld` masquerade |
| IPsec vs SSL VPN tradeoffs | `iptables` NAT table |
| Tunnel establishment & verification | Kernel module management |
| Split tunneling concepts | NetworkManager, sysctl forwarding |

---

## Topology

```
        Nutanix AHV Cluster
    ┌─────────────────────────────────────────────────────────┐
    │                                                         │
    │  ┌─────────────┐         ┌─────────────┐               │
    │  │   Site-A    │         │   Site-B    │               │
    │  │  192.168.   │         │  192.168.   │               │
    │  │   10.0/24   │         │   20.0/24   │               │
    │  │             │         │             │               │
    │  │  ┌─────┐    │         │    ┌─────┐  │               │
    │  │  │HOSTA│    │         │    │HOSTB│  │               │
    │  │  │.10.10    │         │    │.20.20│  │               │
    │  │  └──┬──┘    │         │    └──┬──┘  │               │
    │  │     │       │         │       │     │               │
    │  │  ┌──┴──┐    │         │    ┌──┴──┐  │               │
    │  │  │ GW-A│    │         │    │ GW-B│  │               │
    │  │  │.10.1│    │         │    │.20.1│  │               │
    │  │  │ wg0 │    │         │    │ wg0 │  │               │
    │  │  │.100.1   │         │    │.100.2  │               │
    │  │  └──┬──┘    │         │    └──┬──┘  │               │
    │  └─────┼───────┘         └───────┼─────┘               │
    │        │                         │                     │
    │        └──────────┬──────────────┘                     │
    │                   │                                    │
    │            ┌──────┴──────┐                             │
    │            │  WAN-Transit │  (shared Nutanix network)   │
    │            │ 10.0.0.0/24 │                             │
    │            │  GW-A: .10   │                             │
    │            │  GW-B: .20   │                             │
    │            └─────────────┘                             │
    │                                                         │
    └─────────────────────────────────────────────────────────┘
```

**Key:**
- `wg0` = WireGuard tunnel interface (10.0.100.0/30)
- `WAN-Transit` = shared Nutanix network acting as "the internet"
- Both gateways do NAT for their internal hosts

---

## Prerequisites

- [ ] 4 RHEL VMs: `GW-A`, `GW-B`, `HOSTA`, `HOSTB`
- [ ] 3 Nutanix networks:
  - `Site-A` (192.168.10.0/24, VLAN 110)
  - `Site-B` (192.168.20.0/24, VLAN 120)
  - `WAN-Transit` (10.0.0.0/24, VLAN 130)
- [ ] WireGuard installed on both gateways

> **⚠️ No Prism Admin?** See [NO-PRISM-ADMIN.md](NO-PRISM-ADMIN.md) § Project 3 — WireGuard is a pure overlay and works as-is over your existing flat network. Site subnets move to dummy interfaces; endpoints use existing VM IPs.

> **Air-gapped WireGuard install:**
> ```bash
> # Option A: From RHEL ISO (if available)
> sudo dnf install -y wireguard-tools
>
> # Option B: If not in base repo, use EPEL (pre-staged)
> sudo dnf install -y epel-release
> sudo dnf install -y wireguard-tools
>
> # Option C: Build from source (last resort)
> # Download wireguard-tools tarball on another machine, transfer, compile
> ```

---

## Step 1: Configure Gateway A (GW-A)

### 1.1 Network Interfaces

GW-A has **3 vNICs**: Site-A internal, WAN-Transit, and the WireGuard tunnel.

```bash
# Internal (Site-A)
sudo nmcli con add type ethernet ifname ens3 con-name site-a \
    ipv4.method manual ipv4.addresses 192.168.10.1/24 \
    ipv4.gateway "" ipv4.dns ""

# WAN-Transit (external)
sudo nmcli con add type ethernet ifname ens4 con-name wan \
    ipv4.method manual ipv4.addresses 10.0.0.10/24 \
    ipv4.gateway "" ipv4.dns ""

sudo nmcli con up site-a
sudo nmcli con up wan
```

### 1.2 Enable IP Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/90-ipforward.conf
```

### 1.3 Generate WireGuard Keys

```bash
sudo dnf install -y wireguard-tools

# Generate private key
wg genkey | sudo tee /etc/wireguard/private.key
# Expected: 44-character base64 string

# Generate public key
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
# Expected: 44-character base64 string

# Show public key (share this with GW-B)
sudo cat /etc/wireguard/public.key
```

### 1.4 Create WireGuard Config

```bash
sudo tee /etc/wireguard/wg0.conf << 'EOF'
[Interface]
PrivateKey = <PRIVATE_KEY_FROM_GW-A>
Address = 10.0.100.1/30
ListenPort = 51820

[Peer]
# GW-B public key — fill in after Step 2
PublicKey = <PUBLIC_KEY_FROM_GW-B>
AllowedIPs = 10.0.100.0/30, 192.168.20.0/24
Endpoint = 10.0.0.20:51820
PersistentKeepalive = 25
EOF
```

> **Replace** `<PRIVATE_KEY_FROM_GW-A>` with the actual private key from GW-A.
> **Leave** `<PUBLIC_KEY_FROM_GW-B>` as a placeholder until GW-B is configured.

### 1.5 Start WireGuard

```bash
sudo chmod 600 /etc/wireguard/wg0.conf

# Bring up tunnel
sudo wg-quick up wg0

# Enable at boot
sudo systemctl enable wg-quick@wg0

# Verify
sudo wg show
# Expected: interface wg0, listening port 51820, peer configured
```

### 1.6 Configure NAT (Masquerade)

```bash
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --permanent --zone=public --add-port=51820/udp
sudo firewall-cmd --reload

# Verify NAT table
sudo iptables -t nat -L -v -n
# Expected: MASQUERADE rule in POSTROUTING
```

---

## Step 2: Configure Gateway B (GW-B)

### 2.1 Network Interfaces

```bash
# Internal (Site-B)
sudo nmcli con add type ethernet ifname ens3 con-name site-b \
    ipv4.method manual ipv4.addresses 192.168.20.1/24

# WAN-Transit
sudo nmcli con add type ethernet ifname ens4 con-name wan \
    ipv4.method manual ipv4.addresses 10.0.0.20/24

sudo nmcli con up site-b
sudo nmcli con up wan
```

### 2.2 Enable Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/90-ipforward.conf
```

### 2.3 Generate Keys

```bash
wg genkey | sudo tee /etc/wireguard/private.key
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
sudo cat /etc/wireguard/public.key
# Copy this — you need it for GW-A's config
```

### 2.4 Create WireGuard Config

```bash
sudo tee /etc/wireguard/wg0.conf << 'EOF'
[Interface]
PrivateKey = <PRIVATE_KEY_FROM_GW-B>
Address = 10.0.100.2/30
ListenPort = 51820

[Peer]
# GW-A public key — from Step 1.3
PublicKey = <PUBLIC_KEY_FROM_GW-A>
AllowedIPs = 10.0.100.0/30, 192.168.10.0/24
Endpoint = 10.0.0.10:51820
PersistentKeepalive = 25
EOF
```

> **Replace** both placeholders with actual keys.

### 2.5 Start WireGuard

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0

sudo wg show
# Expected: interface wg0, peer configured
```

### 2.6 Configure NAT

```bash
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --permanent --zone=public --add-port=51820/udp
sudo firewall-cmd --reload
```

---

## Step 3: Exchange Keys and Finalize

### On GW-A

```bash
# Edit config to add GW-B's actual public key
sudo vi /etc/wireguard/wg0.conf
# Replace <PUBLIC_KEY_FROM_GW-B> with the key from GW-B Step 2.3

# Restart tunnel
sudo wg-quick down wg0
sudo wg-quick up wg0

# Verify handshake
sudo wg show
# Expected: latest handshake (recent timestamp), transfer > 0
```

### On GW-B

```bash
# Edit config to add GW-A's actual public key
sudo vi /etc/wireguard/wg0.conf
# Replace <PUBLIC_KEY_FROM_GW-A> with the key from GW-A Step 1.3

sudo wg-quick down wg0
sudo wg-quick up wg0

sudo wg show
# Expected: latest handshake (recent timestamp)
```

---

## Step 4: Configure End Hosts

### HOSTA (Site-A)

```bash
sudo nmcli con add type ethernet ifname ens3 con-name site-a \
    ipv4.method manual ipv4.addresses 192.168.10.10/24 \
    ipv4.gateway 192.168.10.1 ipv4.dns "8.8.8.8"

sudo nmcli con up site-a
sudo hostnamectl set-hostname hosta.lab.local
```

### HOSTB (Site-B)

```bash
sudo nmcli con add type ethernet ifname ens3 con-name site-b \
    ipv4.method manual ipv4.addresses 192.168.20.20/24 \
    ipv4.gateway 192.168.20.1 ipv4.dns "8.8.8.8"

sudo nmcli con up site-b
sudo hostnamectl set-hostname hostb.lab.local
```

---

## Step 5: Validation Tests

### Test 1: Tunnel Interface

```bash
# On GW-A, ping tunnel remote end
ping -c 3 10.0.100.2
# Expected: 3 replies (via WireGuard tunnel)

# On GW-B, ping tunnel remote end
ping -c 3 10.0.100.1
# Expected: 3 replies
```

### Test 2: Cross-Site Connectivity

```bash
# From HOSTA, ping HOSTB through tunnel
ping -c 3 192.168.20.20
# Expected: 3 replies

# Traceroute shows tunnel path
tracepath 192.168.20.20
# Expected: 192.168.10.1 → 10.0.100.2 → 192.168.20.20
```

### Test 3: WireGuard Handshake

```bash
# On either gateway
sudo wg show wg0

# Expected output includes:
# interface: wg0
#   public key: <key>
#   private key: (hidden)
#   listening port: 51820
#
# peer: <peer public key>
#   endpoint: 10.0.0.20:51820 (or 10.0.0.10:51820)
#   allowed ips: 10.0.100.0/30, 192.168.20.0/24 (or 192.168.10.0/24)
#   latest handshake: X seconds ago
#   transfer: X received, X sent
```

### Test 4: NAT Verification

```bash
# On GW-A, check NAT translations
sudo iptables -t nat -L POSTROUTING -v -n

# Expected:
# Chain POSTROUTING (policy ACCEPT ...)
# pkts bytes target     prot opt in     out     source               destination
#   X    X  MASQUERADE  all  --  *      *       192.168.10.0/24     0.0.0.0/0

# From HOSTA, ping WAN-Transit IP of GW-B (bypasses tunnel, tests NAT)
ping -c 3 10.0.0.20
# Expected: replies (source NAT'd to GW-A's WAN IP)
```

### Test 5: Encrypted Traffic

```bash
# On GW-A, capture on WAN interface while pinging HOSTB from HOSTA
sudo tcpdump -i ens4 -n udp port 51820 -c 5

# Expected: UDP packets to/from 10.0.0.20:51820 — WireGuard encrypted payload
# You will NOT see the inner ICMP traffic — it's encrypted
```

---

## CCNA Concepts Mapped

| CCNA Topic | How It's Demonstrated |
|-----------|----------------------|
| Site-to-site VPN | WireGuard tunnel between two gateways |
| NAT/PAT | `firewall-cmd --add-masquerade` on WAN zone |
| Tunnel verification | `wg show` = Cisco `show crypto ipsec sa` |
| Split tunneling | `AllowedIPs` controls what traffic enters tunnel |
| VPN encryption | WireGuard ChaCha20 — all traffic encrypted |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No handshake | `sudo wg show` — verify peer public keys match, endpoint IPs reachable |
| Handshake but no traffic | `AllowedIPs` must include remote subnet; check `ip route` on gateway |
| Can't ping tunnel IP | `wg-quick up wg0` actually started? `ip addr show wg0` |
| NAT not working | `sysctl net.ipv4.ip_forward=1`, `firewall-cmd --list-all` |
| MTU issues | WireGuard MTU default 1420; if fragmentation, lower to 1380 in `[Interface]` |

---

## Cleanup

```bash
# On both gateways
sudo wg-quick down wg0
sudo systemctl disable wg-quick@wg0
sudo rm -f /etc/wireguard/wg0.conf /etc/wireguard/private.key /etc/wireguard/public.key

sudo firewall-cmd --permanent --zone=public --remove-masquerade
sudo firewall-cmd --permanent --zone=public --remove-port=51820/udp
sudo firewall-cmd --reload
```

---

## Next Steps

- Replace WireGuard with **OpenVPN** (SSL VPN) and compare configuration
- Add **dynamic routing** (OSPF) over the tunnel using FRRouting
- Configure **policy-based routing** to send only specific traffic through the tunnel
- Add a **third site** and create a hub-and-spoke VPN topology
