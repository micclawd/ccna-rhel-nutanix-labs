# CCNA + RHEL Lab Projects for Air-Gapped Nutanix (No Prism Admin Required)

Hands-on labs that combine **Cisco CCNA networking concepts** with **Red Hat Enterprise Linux (RHEL) system administration**, designed for an **air-gapped Nutanix AHV environment where you have NO Prism admin rights**.

Every project runs entirely on **existing RHEL VMs with sudo**, on the **existing flat network** your admin already gave you. No creating networks, no uploading ISOs, no adding vNICs, no Prism tickets.

## Projects

| # | Project | VMs Needed | CCNA Skills | RHEL Skills |
|---|---------|-----------|-------------|-------------|
| 1 | [Inter-Subnet Routing & ACLs](project-01-vlan-intervlan-routing.md) | 2-4 | Subnetting, inter-VLAN-style routing, static routes, ACLs | NetworkManager secondary IPs, sysctl forwarding, firewalld rich rules |
| 2 | [RHEL Network Services Stack](project-02-rhel-network-services.md) | 2-3 | DNS, DHCP (DORA), NTP, FTP | bind, dhcpd in network namespaces, chronyd, vsftpd, systemd |
| 3 | [Site-to-Site VPN with WireGuard + NAT](project-03-wireguard-vpn-nat.md) | 2 | VPN tunnels, NAT/PAT, encapsulation | WireGuard, dummy interfaces, firewalld masquerade, conntrack |
| 4 | [High-Availability Web Farm](project-04-ha-web-farm.md) | 2-4 | VRRP/HSRP, VIP failover, load balancing, health checks | keepalived (unicast VRRP), haproxy, httpd |
| 5 | [Centralized Logging + Monitoring](project-05-monitoring-logging.md) | 2-4 | Syslog, SNMP-style monitoring, dashboards | rsyslog remote logging, Prometheus, node_exporter, Grafana (offline) |

## Environment Assumptions

- 2-5 existing **RHEL 8/9 VMs**, each with **1 vNIC** on the same flat network
- **sudo** on every VM
- **Air-gapped**: no internet. Packages already installed, or available via a local repo / mounted RHEL ISO / staged RPMs and tarballs
- You reach VMs via SSH or Prism **console** (console needs no admin rights)

## Conventions Used in Every Guide

```bash
# Your existing NIC and connection (run on each VM first)
DEV=$(ip route show default | awk '{print $5; exit}')
nmcli -f NAME,DEVICE con show --active

# Your existing flat IP
ip -4 addr show scope global
```

- **`<flat-con>`** — your existing NetworkManager connection name (from the command above)
- **Flat IP** — the IP your VM already has. Guides never change it; lab addresses are **added alongside**
- **Lab subnets** — 192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24 (Project 1) and similar. These are *secondary* addresses riding the same wire. If any clash with your flat network, shift them (e.g. 10.10.0.0/24, 10.20.0.0/24) — the guides tell you where

## Shared-Network Etiquette (Read Once, Follow Always)

1. **Never run a DHCP server on the flat network.** Project 2 runs DHCP inside a kernel network namespace — fully isolated, zero risk of poisoning production leases.
2. **Verify before you claim any IP** (e.g. the VRRP virtual IP in Project 4):
   ```bash
   ping -c 2 <candidate-ip>          # expect: silence
   arping -c 2 -D -I $DEV <candidate-ip>   # expect: no replies
   ```
3. **Lab tests must use lab IPs.** Your VMs' flat IPs always reach each other directly — using them in a test bypasses the router/ACL you're trying to verify.

## How to Use

Each project is standalone. Start with Project 1 if you're following the CCNA order, or jump anywhere. Every command is copy-pasteable; every test shows its expected output.
