# Project 2: RHEL Network Services Stack (DNS, DHCP, NTP, FTP)

## Goal

Deploy a single RHEL VM as a centralized network services server, then configure client VMs to consume DNS, DHCP, NTP, and FTP services — all air-gapped.

## Skills Covered

| CCNA | RHEL |
|------|------|
| DNS server operation & port 53 | `bind` / `named` configuration |
| DHCP server operation & ports 67/68 | `dhcpd` configuration |
| NTP stratum & port 123 | `chronyd` configuration |
| FTP active/passive & ports 20/21 | `vsftpd` configuration |
| Client-server verification | `systemd` service management, `firewalld` |

---

## Topology

```
        Nutanix AHV Cluster
    ┌─────────────────────────┐
    │                         │
    │   VLAN100-Services      │
    │   192.168.100.0/24      │
    │        │                │
    │   ┌────┴────┐           │
    │   │  SVCS01 │           │
    │   │ .100.10 │           │
    │   │         │           │
    │   │ DNS     │           │
    │   │ DHCP    │           │
    │   │ NTP     │           │
    │   │ FTP     │           │
    │   └────┬────┘           │
    │        │                │
    │   ┌────┴────┐           │
    │   │  CLI01  │           │
    │   │ .100.20 │           │
    │   └─────────┘           │
    │   ┌─────────┐           │
    │   │  CLI02  │           │
    │   │ .100.21 │           │
    │   └─────────┘           │
    │                         │
    └─────────────────────────┘
```

---

## Prerequisites

- [ ] 3 RHEL VMs: `SVCS01`, `CLI01`, `CLI02`
- [ ] All on the same Nutanix network: `VLAN100-Services` (192.168.100.0/24)
- [ ] SVCS01 has static IP: 192.168.100.10/24
- [ ] Packages staged on SVCS01: `bind`, `dhcp-server`, `chrony`, `vsftpd`

> **Air-gapped tip:** If DNF can't reach repos, mount the RHEL ISO and use it as a local repo:
> ```bash
> sudo mkdir /mnt/rhel-iso
> sudo mount -o loop /path/to/rhel-9.x-x86_64-dvd.iso /mnt/rhel-iso
> sudo cat > /etc/yum.repos.d/rhel-iso.repo << 'EOF'
> [rhel-iso]
> name=RHEL ISO
> baseurl=file:///mnt/rhel-iso/BaseOS
> enabled=1
> gpgcheck=0
> EOF
> sudo dnf install -y bind dhcp-server chrony vsftpd
> ```

---

## Step 1: Configure SVCS01 Base Network

```bash
sudo nmcli con add type ethernet ifname ens3 con-name services \
    ipv4.method manual ipv4.addresses 192.168.100.10/24 \
    ipv4.gateway "" ipv4.dns "127.0.0.1"

sudo nmcli con up services

# Set hostname
sudo hostnamectl set-hostname svcs01.lab.local

# Verify
hostname -f
# Expected: svcs01.lab.local
```

---

## Step 2: Install and Configure DNS (BIND)

### 2.1 Install

```bash
sudo dnf install -y bind bind-utils
```

### 2.2 Configure `/etc/named.conf`

```bash
sudo cp /etc/named.conf /etc/named.conf.bak

sudo tee /etc/named.conf << 'EOF'
options {
    listen-on port 53 { 127.0.0.1; 192.168.100.10; };
    listen-on-v6 port 53 { ::1; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file   "/var/named/data/named.secroots";
    recursing-file  "/var/named/data/named.recursing";
    allow-query     { localhost; 192.168.100.0/24; };
    recursion yes;
    dnssec-validation yes;
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

zone "100.168.192.in-addr.arpa" IN {
    type master;
    file "100.168.192.zone";
    allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
EOF
```

### 2.3 Create Forward Zone

```bash
sudo tee /var/named/lab.local.zone << 'EOF'
$TTL 86400
@   IN  SOA svcs01.lab.local. admin.lab.local. (
        2024091401  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL

    IN  NS  svcs01.lab.local.

svcs01      IN  A   192.168.100.10
fileserver  IN  A   192.168.100.10
ntp         IN  A   192.168.100.10
ftp         IN  A   192.168.100.10
cli01       IN  A   192.168.100.20
cli02       IN  A   192.168.100.21
EOF
```

### 2.4 Create Reverse Zone

```bash
sudo tee /var/named/100.168.192.zone << 'EOF'
$TTL 86400
@   IN  SOA svcs01.lab.local. admin.lab.local. (
        2024091401  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL

    IN  NS  svcs01.lab.local.

10  IN  PTR svcs01.lab.local.
20  IN  PTR cli01.lab.local.
21  IN  PTR cli02.lab.local.
EOF
```

### 2.5 Fix Permissions and Start

```bash
sudo chown root:named /var/named/*.zone
sudo chmod 640 /var/named/*.zone

# Check config
sudo named-checkconf
sudo named-checkzone lab.local /var/named/lab.local.zone
sudo named-checkzone 100.168.192.in-addr.arpa /var/named/100.168.192.zone

# Expected: no errors, "OK" for zone checks

sudo systemctl enable --now named

# Verify listening
sudo ss -tlnp | grep :53
# Expected: named listening on 192.168.100.10:53 and 127.0.0.1:53
```

### 2.6 Open Firewall

```bash
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

---

## Step 3: Install and Configure DHCP

### 3.1 Install

```bash
sudo dnf install -y dhcp-server
```

### 3.2 Configure `/etc/dhcp/dhcpd.conf`

```bash
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.bak

sudo tee /etc/dhcp/dhcpd.conf << 'EOF'
authoritative;
default-lease-time 86400;
max-lease-time 172800;

option domain-name "lab.local";
option domain-name-servers 192.168.100.10;
option ntp-servers 192.168.100.10;

subnet 192.168.100.0 netmask 255.255.255.0 {
    range 192.168.100.100 192.168.100.200;
    option routers 192.168.100.1;
    option broadcast-address 192.168.100.255;
}
EOF

# Test config
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
# Expected: no errors
```

### 3.3 Start and Enable

```bash
sudo systemctl enable --now dhcpd

# Verify listening
sudo ss -ulnp | grep :67
# Expected: dhcpd listening on 0.0.0.0:67

sudo firewall-cmd --permanent --add-service=dhcp
sudo firewall-cmd --reload
```

> **Note:** For this lab, clients will use static IPs to verify DNS/NTP/FTP. DHCP is configured but not actively used by CLI01/CLI02. To test DHCP, create a 4th VM with DHCP enabled.

---

## Step 4: Install and Configure NTP (Chrony)

### 4.1 Install

```bash
sudo dnf install -y chrony
```

### 4.2 Configure `/etc/chrony.conf`

```bash
sudo cp /etc/chrony.conf /etc/chrony.conf.bak

sudo tee /etc/chrony.conf << 'EOF'
# SVCS01 is the NTP master (stratum 10 since air-gapped)
server 127.127.1.0 iburst
local stratum 10

# Allow clients on the local network
allow 192.168.100.0/24

# Serve time even if not synchronized to external source
local stratum 10

driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
```

### 4.3 Start and Enable

```bash
sudo systemctl enable --now chronyd

# Verify
chronyc tracking
# Expected: Reference ID shows 127.127.1.0, Stratum shows 10

sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --reload
```

---

## Step 5: Install and Configure FTP (vsftpd)

### 5.1 Install

```bash
sudo dnf install -y vsftpd
```

### 5.2 Configure `/etc/vsftpd/vsftpd.conf`

```bash
sudo cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak

sudo tee /etc/vsftpd/vsftpd.conf << 'EOF'
anonymous_enable=YES
local_enable=YES
write_enable=YES
local_umask=022
anon_upload_enable=YES
anon_mkdir_write_enable=YES
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen=YES
listen_ipv6=NO
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES

# Passive mode (for firewall traversal)
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
EOF
```

### 5.3 Create Test Content

```bash
sudo mkdir -p /var/ftp/pub
echo "CCNA RHEL Lab Test File" | sudo tee /var/ftp/pub/testfile.txt
sudo chmod 755 /var/ftp/pub
sudo chmod 644 /var/ftp/pub/testfile.txt

# Create local user for FTP
sudo useradd -d /home/ftpuser -s /sbin/nologin ftpuser
echo "ftpuser:LabPass123" | sudo chpasswd
```

### 5.4 Start and Enable

```bash
sudo systemctl enable --now vsftpd

sudo firewall-cmd --permanent --add-service=ftp
sudo firewall-cmd --permanent --add-port=40000-40100/tcp
sudo firewall-cmd --reload

# Verify
sudo ss -tlnp | grep :21
# Expected: vsftpd listening on :::21 or 0.0.0.0:21
```

---

## Step 6: Configure Clients

### CLI01 (192.168.100.20)

```bash
sudo nmcli con add type ethernet ifname ens3 con-name lab \
    ipv4.method manual ipv4.addresses 192.168.100.20/24 \
    ipv4.gateway "" ipv4.dns "192.168.100.10"

sudo nmcli con up lab

sudo hostnamectl set-hostname cli01.lab.local
```

### CLI02 (192.168.100.21)

```bash
sudo nmcli con add type ethernet ifname ens3 con-name lab \
    ipv4.method manual ipv4.addresses 192.168.100.21/24 \
    ipv4.gateway "" ipv4.dns "192.168.100.10"

sudo nmcli con up lab

sudo hostnamectl set-hostname cli02.lab.local
```

---

## Step 7: Validation Tests

### Test 1: DNS Resolution

```bash
# From CLI01, resolve forward lookup
nslookup svcs01.lab.local
# Expected: Server: 192.168.100.10, Address: 192.168.100.10#53
#           Name: svcs01.lab.local, Address: 192.168.100.10

nslookup fileserver.lab.local
# Expected: resolves to 192.168.100.10

# Reverse lookup
nslookup 192.168.100.10
# Expected: name = svcs01.lab.local

# Test from SVCS01 itself
dig @localhost lab.local ANY
```

### Test 2: NTP Synchronization

```bash
# On CLI01, point to SVCS01
sudo tee /etc/chrony.conf << 'EOF'
server 192.168.100.10 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
EOF

sudo systemctl restart chronyd

# Wait 10 seconds, then check
chronyc sources -v
# Expected: ^* svcs01.lab.local ... with reach 377

chronyc tracking
# Expected: Reference ID showing SVCS01 IP
```

### Test 3: FTP Download

```bash
# From CLI01, anonymous FTP
ftp ftp://192.168.100.10/pub/testfile.txt
# Expected: file downloads, contents shown

# Or with curl
curl -s ftp://192.168.100.10/pub/testfile.txt
# Expected: "CCNA RHEL Lab Test File"

# With local user
curl -s --user ftpuser:LabPass123 ftp://192.168.100.10/
# Expected: directory listing
```

### Test 4: DHCP (Optional)

Create a 4th VM with DHCP enabled:

```bash
# On the new VM
sudo nmcli con add type ethernet ifname ens3 con-name dhcp-client \
    ipv4.method auto

sudo nmcli con up dhcp-client

# Check lease
nmcli con show dhcp-client | grep ipv4
# Expected: IP in 192.168.100.100-200 range, DNS = 192.168.100.10

# Verify lease on server
sudo cat /var/lib/dhcpd/dhcpd.leases
```

---

## CCNA Concepts Mapped

| CCNA Topic | How It's Demonstrated |
|-----------|----------------------|
| DNS | BIND authoritative + recursive, forward/reverse zones |
| DHCP | dhcpd subnet declaration, options (DNS, NTP, router) |
| NTP | Chrony as stratum 10 master, client sync |
| FTP | vsftpd active/passive, anonymous + authenticated |
| Client verification | `nslookup`, `chronyc`, `ftp` commands |
| Port numbers | 53 (DNS), 67/68 (DHCP), 123 (NTP), 20/21 (FTP) |

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| DNS not resolving | `named-checkconf`, `named-checkzone`, `firewall-cmd --list-all` |
| DHCP no leases | `dhcpd -t`, `journalctl -u dhcpd`, ensure VM NIC on correct Nutanix network |
| NTP not syncing | `chronyc sources`, check `allow` in server config, UDP 123 open |
| FTP connection refused | `systemctl status vsftpd`, check passive ports in firewall |
| FTP hangs after login | Passive ports blocked; add 40000-40100/tcp to firewall |

---

## Cleanup

```bash
# Stop all services
sudo systemctl stop named dhcpd chronyd vsftpd

# Remove packages (optional)
sudo dnf remove -y bind dhcp-server chrony vsftpd

# Delete zones/configs
sudo rm -f /var/named/lab.local.zone /var/named/100.168.192.zone
```

---

## Next Steps

- Add **DNS forwarders** to simulate internet DNS
- Configure **DHCP failover** between two servers
- Set up **SFTP** (SSH File Transfer) as a secure alternative
- Add **SNMP** monitoring to SVCS01
