# CCNA + RHEL Lab Projects for Air-Gapped Nutanix

A collection of hands-on lab projects that combine **Cisco CCNA networking concepts** with **Red Hat Enterprise Linux (RHEL) system administration**, designed to run entirely inside an **air-gapped Nutanix AHV cluster**.

## Projects

| # | Project | CCNA Skills | RHEL Skills |
|---|---------|-------------|-------------|
| 1 | [VLAN-Isolated Multi-Site Network](project-01-vlan-intervlan-routing.md) | VLANs, 802.1Q, inter-VLAN routing, static routes, ACLs | NetworkManager, firewall-cmd, sysctl, ip_forward |
| 2 | [RHEL Network Services Stack](project-02-rhel-network-services.md) | DNS, DHCP, NTP, FTP server roles | bind, dhcpd, chronyd, vsftpd, systemd, firewalld |
| 3 | [Site-to-Site VPN with WireGuard + NAT](project-03-wireguard-vpn-nat.md) | VPN concepts, NAT/PAT, tunneling | WireGuard, firewalld masquerade, iptables, kernel modules |
| 4 | [High-Availability Web Farm](project-04-ha-web-farm.md) | Load balancing, redundancy, VRRP/HSRP concepts | keepalived, haproxy, httpd, systemd dependencies |
| 5 | [Centralized Logging + Monitoring](project-05-monitoring-logging.md) | SNMP, syslog, network monitoring | rsyslog, Prometheus, node_exporter, Grafana |

## Prerequisites

- Nutanix AHV cluster with Prism access
- RHEL 9.x ISO uploaded to Prism > Images
- No internet access (air-gapped) — all packages must be pre-staged on a golden image
- Console access to VMs via Prism

> **⚠️ No Prism admin rights?** Every project has a flat-network variant needing zero Prism changes — see [NO-PRISM-ADMIN.md](NO-PRISM-ADMIN.md) for the full adaptation guide (secondary subnets, network-namespace DHCP sandbox, unicast VRRP, dummy-interface site subnets).

## How to Use

Each project is a standalone markdown guide. Start with Project 1 and work through in order, or jump to any project that matches your current study focus.

## Contributing

These guides were built for a specific lab environment. Adapt IP ranges, VLAN IDs, and hostnames to match your own setup.
