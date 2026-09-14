# Running These Labs WITHOUT Prism Admin Rights

You don't need Prism admin to do any of these projects. Every lab has a **flat-network variant** that runs entirely inside RHEL on VMs you already have. This file is the master adaptation guide; each project file also has a short "⚠️ No Prism Admin?" section inline.

---

## Assumptions

- You have **2-5 RHEL VMs**, each with **1 vNIC**, all on the **same existing flat network** (whatever subnet your admin already put you on)
- You have **sudo** on each VM
- You **cannot** create networks, upload ISOs, add vNICs, or change VM specs in Prism
- Multicast on the flat network is **unknown / possibly filtered** (matters for VRRP — solved below)
- Packages are already installed, or you have a local repo / staged RPMs

## Golden Rules (Shared Network Safety)

1. **Never run a DHCP server on the shared network** — rogue DHCP will hand out leases to production VMs and your admin will (rightly) hunt you down. Use the namespace sandbox in Project 2.
2. **Never claim an IP that's in use** — before using any address (VIP, secondary IP), verify: `ping -c 2 <ip>` returns nothing AND `arping -c 2 -D <ip>` shows no owner. When in doubt, ask the admin for a small reserved block.
3. **"VLANs" become "subnets on one wire"** — you lose L2 broadcast isolation, but every routing, ACL, and services skill is still practiced.
4. **Prefer unicast VRRP** — don't depend on multicast you can't verify.
5. **Overlays need zero network changes** — WireGuard (Project 3) works perfectly as-is.

---

## Project 1 — Replace VLANs with Secondary Subnets

**The trick:** one vNIC can carry many IP subnets. Your routers stack 3 subnet addresses on one interface; hosts point their gateway at the router's address in their own subnet. Traffic hairpins through the router — which is exactly the inter-VLAN routing behavior you're practicing.

### Router R1 (single vNIC)

```bash
# Find your existing connection name
nmcli -f NAME,DEVICE con show --active

# Stack the three subnet gateway IPs onto it
sudo nmcli con mod "<existing-connection>" \
    +ipv4.addresses "192.168.10.1/24" \
    +ipv4.addresses "192.168.20.1/24" \
    +ipv4.addresses "192.168.30.1/24"

sudo nmcli con up "<existing-connection>"

# Disable ICMP redirects (cleaner hairpin routing)
sudo sysctl -w net.ipv4.conf.all.send_redirects=0
echo "net.ipv4.conf.all.send_redirects=0" | sudo tee -a /etc/sysctl.d/90-ipforward.conf

# Enable forwarding (same as main guide)
sudo sysctl -w net.ipv4.ip_forward=1
```

**Verify:**
```bash
ip -4 addr show ens3 | grep 192.168
# Expected:
# inet 192.168.10.1/24 ...
# inet 192.168.20.1/24 ...
# inet 192.168.30.1/24 ...
```

### Hosts

Identical to the main guide's Step 4 — USER01 gets `192.168.10.10/24` with gateway `192.168.10.1`, etc. The only difference: all VMs physically share the flat network instead of sitting on separate VLANs.

### R2

Optional. Either skip it (single-router variant — still covers every listed skill), or stack `192.168.x.254` addresses the same way and use another pair of secondary addresses (`10.0.99.1/30` / `10.0.99.2/30`) as the transit link.

### firewalld ACLs

**Unchanged.** Everything in Step 2.4 works as written — rich rules match on IP, not on physical topology. DMZ→Users rejects, DMZ→Servers:80 permits, all verified with the same test commands.

### What you lose vs. real VLANs

- No true broadcast-domain isolation (all VMs see the same wire)
- No 802.1Q tagging practice on the switch side — if you want that, ask your admin for **one trunked vNIC** (see "What to Ask Your Admin For" below)

---

## Project 2 — DHCP in a Network Namespace Sandbox

DNS, NTP, FTP: run exactly as the main guide says, on the flat network. Clients point at SVCS01's existing IP.

**DHCP is the problem child.** Running `dhcpd` on a shared production network is how you become the office villain. Solution: build a completely isolated L2 segment **inside SVCS01** using network namespaces — the same kernel technology containers are built on. Zero Prism changes, zero risk to the network, and honestly better RHEL practice.

### Build the sandbox

```bash
# Two namespaces: one runs dhcpd, one runs the client
sudo ip netns add dhcpserver
sudo ip netns add dhcpclient

# veth pair = virtual patch cable between them
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth0 netns dhcpserver
sudo ip link set veth1 netns dhcpclient

# Loopbacks up inside each namespace
sudo ip netns exec dhcpserver ip link set lo up
sudo ip netns exec dhcpclient ip link set lo up

# Server side: static IP on the sandbox subnet
sudo ip netns exec dhcpserver ip addr add 10.77.0.1/24 dev veth0
sudo ip netns exec dhcpserver ip link set veth0 up

# Client side: interface up, NO address (DHCP will provide it)
sudo ip netns exec dhcpclient ip link set veth1 up
```

### Server config (sandbox-only)

```bash
sudo tee /etc/dhcp/dhcpd-ns.conf << 'EOF'
authoritative;
default-lease-time 3600;
max-lease-time 7200;

subnet 10.77.0.0 netmask 255.255.255.0 {
    range 10.77.0.100 10.77.0.200;
    option routers 10.77.0.1;
    option domain-name-servers 10.77.0.1;
    option domain-name "sandbox.local";
}
EOF

# Validate syntax BEFORE starting
sudo dhcpd -t -cf /etc/dhcp/dhcpd-ns.conf
# Expected: no output (clean) or "Configuration file: ..." with no errors
```

### Run the server and client

```bash
# Install client tools if needed
sudo dnf install -y dhcp-client

# Start dhcpd INSIDE the namespace, bound to veth0 only
sudo ip netns exec dhcpserver dhcpd -cf /etc/dhcp/dhcpd-ns.conf \
    -lf /var/lib/dhcpd/dhcpd-ns.leases veth0

# Request a lease from the client namespace
sudo ip netns exec dhcpclient dhclient -v veth1
```

**Verify:**
```bash
sudo ip netns exec dhcpclient ip addr show veth1
# Expected: inet 10.77.0.100/24 (or anything in .100-.200)

sudo cat /var/lib/dhcpd/dhcpd-ns.leases
# Expected: a lease block with the assigned IP and client MAC

# Full end-to-end: client pings server
sudo ip netns exec dhcpclient ping -c 3 10.77.0.1
# Expected: 3 replies
```

### Cleanup

```bash
sudo ip netns exec dhcpclient dhclient -r veth1 2>/dev/null
sudo pkill -f "dhcpd.*dhcpd-ns" 2>/dev/null
sudo ip netns del dhcpserver
sudo ip netns del dhcpclient
```

> **CCNA note:** everything you'd verify on real DHCP — DORA exchange, lease file, scope options — is present here. Run `sudo ip netns exec dhcpserver tcpdump -i veth0 -n port 67 or port 68` during the client request to **watch the Discover/Offer/Request/ACK exchange live**. That's the whole CCNA DHCP lesson in one tcpdump.

---

## Project 3 — WireGuard Is Pure Overlay

**No changes needed to the core guide.** WireGuard tunnels over whatever network already exists — your flat network IS the "WAN-Transit."

The only adaptation: site subnets live on **dummy interfaces** instead of separate host VMs.

### Site subnet on a dummy interface

```bash
# On GW-A
sudo nmcli con add type dummy ifname site0 \
    ipv4.method manual ipv4.addresses 192.168.10.1/24
sudo nmcli con up site0

# On GW-B
sudo nmcli con add type dummy ifname site0 \
    ipv4.method manual ipv4.addresses 192.168.20.1/24
sudo nmcli con up site0
```

### WireGuard endpoints

In `wg0.conf`, set `Endpoint` to the **other VM's existing flat-network IP** (e.g. `Endpoint = 172.205.2.51:51820`). Make sure each VM's firewalld allows UDP 51820 — the guide already covers this.

### Testing without HOSTA/HOSTB

```bash
# From GW-A, ping GW-B's site subnet through the tunnel
ping -c 3 192.168.20.1
# Expected: 3 replies — packet path: wg0 → encrypted UDP → flat network → GW-B wg0 → dummy

# Confirm it went through the tunnel, not the flat network directly
tracepath 192.168.20.1
# Expected: first hop is 10.0.100.2 (tunnel), NOT the flat gateway
```

### NAT demo without extra VMs

```bash
# On GW-A: masquerade on the flat interface
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --reload

# Ping another VM on the flat network, sourced FROM the site subnet
ping -I 192.168.10.1 -c 4 <some-other-flat-vm-ip>

# Watch NAT counters climb
sudo iptables -t nat -L POSTROUTING -v -n
# Expected: pkts/bytes incrementing on the MASQUERADE rule

# See live translations
sudo cat /proc/net/nf_conntrack | grep 192.168.10.1 | head -5
```

---

## Project 4 — Unicast VRRP on a Shared Network

Two adaptations, both mandatory on a shared network.

### 1. VIP discipline

The virtual IP must be a **free IP on your flat subnet**:

```bash
# Verify candidate is unused
ping -c 2 <candidate-vip>
# Expected: no replies

arping -c 2 -D -I ens3 <candidate-vip>
# Expected: no replies (no one owns this MAC/IP pair)
```

Best practice: ask your admin for one reserved IP. If you can't, pick something high in the range (e.g. `.250+`) after the checks above, and be ready to release it.

### 2. Unicast VRRP (multicast may be filtered)

Add to `vrrp_instance` in `keepalived.conf` on **both** load balancers:

```
vrrp_instance VI_1 {
    state MASTER              # BACKUP on LB02
    interface ens3
    unicast_src_ip <this-lb-flat-IP>
    unicast_peer {
        <other-lb-flat-IP>
    }
    virtual_router_id 51
    priority 150              # 100 on LB02
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LabPass123
    }
    virtual_ipaddress {
        <free-flat-IP>/24
    }
}
```

### Short on VMs?

Collapse to **2 VMs**: LB01+WEB01 on one, LB02+WEB02 on the other. HAProxy on each box load-balances across both web servers; Keepalived floats the VIP between them. Failover, health checks, and round-robin all still fully demonstrable.

Validation is identical to the main guide: `curl http://<vip>/` in a loop, stop `httpd` on one box, stop `keepalived` on the master, watch the VIP move (`ip addr show ens3 | grep <vip>` on the backup).

---

## Project 5 — Works As-Is

Zero Prism dependencies already. All VMs use their existing flat IPs. Two air-gapped notes:

### Grafana dashboards without internet

`Import dashboard ID 1860` requires Grafana to reach grafana.com. Offline alternative — provision from local JSON:

```bash
sudo mkdir -p /var/lib/grafana/dashboards /opt/grafana/conf/provisioning/dashboards

sudo tee /opt/grafana/conf/provisioning/dashboards/local.yaml << 'EOF'
apiVersion: 1
providers:
  - name: local
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
EOF

sudo chown -R grafana:grafana /var/lib/grafana/dashboards
```

On any machine with internet, download the Node Exporter Full dashboard JSON (grafana.com/dashboards/1860 → "Download JSON"), transfer it to MON01, drop it in `/var/lib/grafana/dashboards/`, and restart Grafana. It appears automatically.

### Verify everything without a browser

```bash
# Prometheus targets
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep '"health"'

# Actual metrics flowing
curl -s 'http://localhost:9090/api/v1/query?query=up' | python3 -m json.tool

# Grafana data source
curl -s http://admin:admin@localhost:3000/api/datasources | python3 -m json.tool | grep -E '"name"|"type"'
```

---

## What to Ask Your Prism Admin For (One-Time Email)

If you can get even a few of these, the labs get closer to the "real" versions. Copy-paste friendly:

> Hi, I'm building some RHEL networking labs for CCNA/RHEL study. Could I get:
>
> 1. **4 RHEL 9 VMs** (2 vCPU / 4 GB RAM / 40 GB disk each) on `<our usual network>` — or confirmation I can use my existing ones
> 2. **Console and/or SSH access** to all of them
> 3. *(Optional, enables VLAN labs)* **A second vNIC on 2 of the VMs**, on any isolated VLAN
> 4. *(Optional, enables real DHCP/routing labs)* **One isolated VLAN network** (no DHCP from your side, no gateway needed) for lab use
> 5. *(Optional)* **A small reserved IP block** (e.g. 5 addresses) on our flat network for VRRP VIP testing
>
> Nothing here touches production networking; all services run inside the VMs except items 3-5.

If you get item 4, Project 1 and Project 2 become fully "real" — true L2 isolation and safe live DHCP.

---

## Lab Variant Matrix

| Project | Full Version Needs | Flat-Network Variant Needs | Skills Lost in Variant |
|---------|-------------------|---------------------------|------------------------|
| 1. VLAN routing | 3 VLAN networks + multi-vNIC VMs | Nothing (secondary subnets) | 802.1Q switch-side, true L2 isolation |
| 2. Services | Nothing (1 network) | Nothing — DHCP moves into namespaces | None |
| 3. WireGuard VPN | 3 networks (nice-to-have) | Nothing (pure overlay) | None |
| 4. HA web farm | Nothing (1 network) | Free IP for VIP + unicast VRRP | None |
| 5. Monitoring | Nothing (1 network) | Nothing | None |
