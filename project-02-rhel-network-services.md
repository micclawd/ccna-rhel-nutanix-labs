# Project 2: RHEL Network Services Stack (DNS, DHCP, NTP, FTP)

**No Prism admin required.** Runs on your existing VMs, on your existing flat network.

## Goal

Turn one RHEL VM into a full network-services server — DNS, NTP, FTP live on the flat network; DHCP runs in an **isolated kernel network namespace** so you can study the DORA exchange without poisoning production leases.

## Skills Covered

| CCNA | RHEL |
|------|------|
| DNS operation, port 53, forward/reverse zones | `bind` / `named` configuration |
| DHCP DORA exchange, ports 67/68 | `dhcpd` inside `ip netns` sandbox |
| NTP stratum, port 123 | `chronyd` as local master |
| FTP active/passive, ports 20/21 | `vsftpd` configuration |
| Client verification | `nmcli`, `chronyc`, `curl`, `tcpdump` |

---

## Topology

```
        Existing flat network
    ┌────────────────────────────────────┐
    │                                    │
    │  ┌───────────┐   ┌───────────┐     │
    │  │  SVCS01   │   │  CLI01    │     │
    │  │ (flat IP) │   │ (flat IP) │     │
    │  │           │   │           │     │
    │  │ DNS :53   │   │ uses      │     │
    │  │ NTP :123  │   │ SVCS01    │     │
    │  │ FTP :21   │   │ for all   │     │
    │  │           │   │           │     │
    │  │ ┌───────┐ │   └───────────┘     │
    │  │ │netns  │ │                     │
    │  │ │DHCP   │ │  (isolated inside   │
    │  │ │10.77.0│ │   SVCS01 — never    │
    │  │ └───────┘ │   touches the wire) │
    │  └───────────┘                     │
    └────────────────────────────────────┘
```

---

## Prerequisites

- [ ] 2 existing RHEL VMs (SVCS01, CLI01); a 3rd (CLI02) optional
- [ ] sudo on both
- [ ] Packages available: `bind bind-utils chrony vsftpd dhcp-server dhcp-client tcpdump`

**Air-gapped package install** — mount the RHEL ISO (ask whoever built the VMs to attach it, or reuse one already attached) as a local repo:

```bash
sudo mkdir -p /mnt/rhel-iso
sudo mount -o loop /path/to/rhel-9.x-x86_64-dvd.iso /mnt/rhel-iso   # or: sudo mount /dev/sr0 /mnt/rhel-iso

sudo tee /etc/yum.repos.d/rhel-iso.repo << 'EOF'
[rhel-iso-baseos]
name=RHEL ISO BaseOS
baseurl=file:///mnt/rhel-iso/BaseOS
enabled=1
gpgcheck=0

[rhel-iso-appstream]
name=RHEL ISO AppStream
baseurl=file:///mnt/rhel-iso/AppStream
enabled=1
gpgcheck=0
EOF

sudo dnf --disablerepo='*' --enablerepo='rhel-iso-*' install -y \
    bind bind-utils chrony vsftpd dhcp-server dhcp-client tcpdump
```

---

## Step 1: Identify SVCS01's Address

Everything in this guide is built around the **flat IP SVCS01 already has**.

```bash
# On SVCS01
SVCS01_IP=$(ip -4 -o addr show scope global | awk 'NR==1{split($4,a,"/");print a[1]}')
DEV=$(ip route show default | awk '{print $5; exit}')
echo "$SVCS01_IP on $DEV"
# Expected: your flat IP, e.g. 172.205.2.51 on ens3
# WRITE THIS IP DOWN — CLI01 needs it in Step 6

sudo hostnamectl set-hostname svcs01.lab.local
hostname -f
# Expected: svcs01.lab.local
```

> **Placeholders below:** configs are written with `__IP__` / `__REV__` / `__LAST__` tokens, then `sed` injects your real values. Copy-paste safe for any flat subnet.

---

## Step 2: DNS (BIND)

### 2.1 Main Config

```bash
sudo cp /etc/named.conf /etc/named.conf.bak 2>/dev/null

sudo tee /etc/named.conf << 'EOF'
options {
    listen-on port 53 { 127.0.0.1; __IP__; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file   "/var/named/data/named.secroots";
    recursing-file  "/var/named/data/named.recursing";
    allow-query     { localhost; any; };
    recursion yes;
    dnssec-validation no;
    managed-keys-directory "/var/named/dynamic";
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
};

zone "lab.local" IN {
    type master;
    file "lab.local.zone";
    allow-update { none; };
};

zone "__REV__.in-addr.arpa" IN {
    type master;
    file "reverse.zone";
    allow-update { none; };
};

include "/etc/named.rfc1912.zones";
EOF

sudo sed -i "s|__IP__|$SVCS01_IP|g" /etc/named.conf
```

### 2.2 Zone Files

```bash
OCTETS=$(echo "$SVCS01_IP" | cut -d. -f1-3)
REV=$(echo "$SVCS01_IP" | awk -F. '{print $3"."$2"."$1}')
LAST=$(echo "$SVCS01_IP" | awk -F. '{print $4}')

sudo sed -i "s|__REV__|$REV|g" /etc/named.conf

sudo tee /var/named/lab.local.zone << EOF
\$TTL 86400
@   IN  SOA svcs01.lab.local. admin.lab.local. (
        2024091401  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL

    IN  NS  svcs01.lab.local.

svcs01      IN  A   $SVCS01_IP
dns         IN  CNAME   svcs01
ntp         IN  CNAME   svcs01
ftp         IN  CNAME   svcs01
EOF

sudo tee /var/named/reverse.zone << EOF
\$TTL 86400
@   IN  SOA svcs01.lab.local. admin.lab.local. (
        2024091401 3600 1800 604800 86400 )

    IN  NS  svcs01.lab.local.

$LAST   IN  PTR svcs01.lab.local.
EOF

sudo chown root:named /var/named/lab.local.zone /var/named/reverse.zone
sudo chmod 640 /var/named/lab.local.zone /var/named/reverse.zone
```

### 2.3 Validate and Start

```bash
sudo named-checkconf
# Expected: no output = clean

sudo named-checkzone lab.local /var/named/lab.local.zone
# Expected: zone lab.local/IN: loaded serial ... OK

sudo named-checkzone "$REV.in-addr.arpa" /var/named/reverse.zone
# Expected: ... OK

sudo systemctl enable --now named
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload

ss -tlnp | grep :53
# Expected: named on __IP__:53 and 127.0.0.1:53
```

---

## Step 3: NTP (Chrony) — Stratum 10 Local Master

Air-gapped means no upstream NTP. Chrony's `local` directive makes SVCS01 an authoritative source anyway.

```bash
sudo cp /etc/chrony.conf /etc/chrony.conf.bak

sudo tee /etc/chrony.conf << 'EOF'
# Air-gapped master: serve local clock at stratum 10
local stratum 10
allow all

driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF

sudo systemctl enable --now chronyd
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --reload

chronyc tracking
# Expected: Stratum: 10 (after a few seconds)
```

---

## Step 4: FTP (vsftpd)

```bash
sudo cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak

sudo tee /etc/vsftpd/vsftpd.conf << 'EOF'
anonymous_enable=YES
local_enable=YES
write_enable=YES
local_umask=022
anon_upload_enable=NO
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen=YES
listen_ipv6=NO
pam_service_name=vsftpd
userlist_enable=YES
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
EOF

sudo mkdir -p /var/ftp/pub
echo "CCNA RHEL Lab Test File - $(hostname)" | sudo tee /var/ftp/pub/testfile.txt
sudo chmod 755 /var/ftp/pub

sudo systemctl enable --now vsftpd
sudo firewall-cmd --permanent --add-service=ftp
sudo firewall-cmd --permanent --add-port=40000-40100/tcp
sudo firewall-cmd --reload

ss -tlnp | grep :21
# Expected: vsftpd listening
```

---

## Step 5: DHCP — In an Isolated Namespace Sandbox

**Never run dhcpd on the flat interface.** Instead, build a virtual patch cable (veth pair) between two kernel network namespaces *inside SVCS01* and run the whole exchange there. Same protocol, same lease file, zero blast radius.

### 5.1 Build the Sandbox

```bash
sudo ip netns add dhcpserver
sudo ip netns add dhcpclient

sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth0 netns dhcpserver
sudo ip link set veth1 netns dhcpclient

sudo ip netns exec dhcpserver ip link set lo up
sudo ip netns exec dhcpclient ip link set lo up

sudo ip netns exec dhcpserver ip addr add 10.77.0.1/24 dev veth0
sudo ip netns exec dhcpserver ip link set veth0 up
sudo ip netns exec dhcpclient ip link set veth1 up

sudo ip netns exec dhcpserver ip addr show veth0
# Expected: inet 10.77.0.1/24
```

### 5.2 Server Config

```bash
sudo mkdir -p /var/lib/dhcpd

sudo tee /etc/dhcp/dhcpd-ns.conf << 'EOF'
authoritative;
default-lease-time 3600;
max-lease-time 7200;

option domain-name "sandbox.local";
option domain-name-servers 10.77.0.1;

subnet 10.77.0.0 netmask 255.255.255.0 {
    range 10.77.0.100 10.77.0.200;
    option routers 10.77.0.1;
}
EOF

sudo dhcpd -t -cf /etc/dhcp/dhcpd-ns.conf
# Expected: clean exit, no errors
```

### 5.3 Run the Exchange (and Watch DORA Live)

Open **two shells** on SVCS01:

```bash
# SHELL 1 — start packet capture inside the server namespace
sudo ip netns exec dhcpserver tcpdump -i veth0 -n -v port 67 or port 68

# SHELL 2 — start dhcpd, then request a lease from the client namespace
sudo ip netns exec dhcpserver dhcpd -cf /etc/dhcp/dhcpd-ns.conf \
    -lf /var/lib/dhcpd/dhcpd-ns.leases veth0

sudo ip netns exec dhcpclient dhclient -v veth1
```

**Shell 1 expected output — the full CCNA DORA lesson:**
```
... DHCP Discover ... from 0.0.0.0.bootpc > 255.255.255.255.bootps
... DHCP Offer ... 10.77.0.100
... DHCP Request ...
... DHCP Ack ...
```

**Verify the lease:**
```bash
sudo ip netns exec dhcpclient ip addr show veth1
# Expected: inet 10.77.0.100/24 (or anything in .100-.200)

sudo cat /var/lib/dhcpd/dhcpd-ns.leases
# Expected: lease block with IP, client MAC, start/end times

sudo ip netns exec dhcpclient ping -c 3 10.77.0.1
# Expected: 3 replies
```

### 5.4 Cleanup the Sandbox (when done)

```bash
sudo ip netns exec dhcpclient dhclient -r veth1 2>/dev/null
sudo pkill -f 'dhcpd.*dhcpd-ns' 2>/dev/null
sudo ip netns del dhcpserver
sudo ip netns del dhcpclient
```

---

## Step 6: Configure the Client (CLI01)

```bash
# On CLI01
DEV=$(ip route show default | awk '{print $5; exit}')
nmcli -f NAME,DEVICE con show --active   # note <flat-con>

sudo hostnamectl set-hostname cli01.lab.local

# Point DNS at SVCS01 (type SVCS01's flat IP literally)
sudo nmcli con mod "<flat-con>" ipv4.dns "<SVCS01_FLAT_IP>"
sudo nmcli con up "<flat-con>"

# Point NTP at SVCS01
sudo cp /etc/chrony.conf /etc/chrony.conf.bak
sudo tee /etc/chrony.conf << 'EOF'
server <SVCS01_FLAT_IP> iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
EOF
sudo sed -i "s|<SVCS01_FLAT_IP>|<SVCS01_FLAT_IP>|" /etc/chrony.conf
sudo systemctl restart chronyd
```

---

## Step 7: Validation Tests (from CLI01)

### DNS

```bash
nslookup svcs01.lab.local
# Expected: Server: <SVCS01_FLAT_IP> ... Name: svcs01.lab.local  Address: <SVCS01_FLAT_IP>

nslookup ftp.lab.local
# Expected: ftp.lab.local canonical name = svcs01.lab.local

nslookup <SVCS01_FLAT_IP>
# Expected: name = svcs01.lab.local (reverse lookup works)

dig @<SVCS01_FLAT_IP> lab.local ANY
# Expected: SOA + NS + A records, status: NOERROR
```

### NTP

```bash
chronyc sources -v
# Expected: ^* svcs01.lab.local (or the IP) with Reach: 377 (wait ~30s after restart)

chronyc tracking
# Expected: Reference ID = SVCS01's IP, Stratum: 11 (one below the master)
```

### FTP

```bash
curl -s ftp://<SVCS01_FLAT_IP>/pub/testfile.txt
# Expected: CCNA RHEL Lab Test File - svcs01.lab.local
```

---

## CCNA Concepts Mapped

| CCNA Topic | Where You Did It |
|-----------|------------------|
| DNS forward/reverse lookup | lab.local zone + reverse zone, `nslookup` tests |
| DHCP DORA | Live `tcpdump` of Discover/Offer/Request/Ack in the namespace sandbox |
| DHCP scope & options | `subnet` block: range, routers, domain-name-servers |
| NTP stratum | SVCS01 stratum 10 master; CLI01 stratum 11 client |
| FTP passive mode | `pasv_min/max_port` + firewall range — the CCNA "FTP uses two ports" gotcha |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| named won't start | `sudo named-checkconf -z` and `journalctl -u named -n 30 --no-pager` |
| DNS times out from client | `sudo firewall-cmd --list-all` on SVCS01 — dns service present? Correct zone on the flat interface? |
| dhclient gets nothing | Is dhcpd actually running in the namespace? `sudo ip netns exec dhcpserver pgrep -a dhcpd`. tcpdump shows Discover but no Offer → check `dhcpd -t` syntax |
| chrony never selects source | `allow all` present in SVCS01 chrony.conf? UDP 123 open? `sudo ss -ulnp \| grep 123` |
| FTP login OK, listing hangs | Passive ports blocked — confirm `--add-port=40000-40100/tcp` and `--reload` |
| Zone transfer/serial errors | Bump the serial number in the zone file every edit, then `sudo systemctl reload named` |

---

## Cleanup

```bash
sudo systemctl stop named chronyd vsftpd
sudo firewall-cmd --permanent --remove-service=dns
sudo firewall-cmd --permanent --remove-service=ntp
sudo firewall-cmd --permanent --remove-service=ftp
sudo firewall-cmd --permanent --remove-port=40000-40100/tcp
sudo firewall-cmd --reload
sudo rm -f /var/named/lab.local.zone /var/named/reverse.zone /etc/dhcp/dhcpd-ns.conf
# Namespace sandbox cleanup: see Step 5.4
```

---

## Next Steps

- Add a **second zone** (e.g. `branch.lab.local`) and a conditional forwarder between two BIND servers
- Give the namespace DHCP client **internet-less routing** via SVCS01 (`ip netns` + NAT) to study default-gateway behavior
- Project 5 adds monitoring — point node_exporter at SVCS01 and watch DNS query load in Grafana
