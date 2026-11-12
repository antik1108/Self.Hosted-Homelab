# Current Network Configuration (Live State)

> **Generated**: 2026-11-12 | **Source**: Live device inspection

---

## OpenWrt Router (TP-Link Archer C20 v5)

### System
```
Model: TP-Link Archer C20 v5
Architecture: ramips/mt76x8
Firmware: OpenWrt 25.12.6 (SNAPSHOT, r0-0000000000)
Kernel: 6.6.86
Overlay: 960 KB (93% used, 72 KB free)
RAM: 128 MB (45 MB free)
Uptime: 2h 15m
```

### Network Interfaces (UCI)
```bash
# /etc/config/network
config interface 'loopback'
    option device 'lo'
    option proto 'static'
    option ipaddr '127.0.0.1'
    option netmask '255.0.0.0'

config globals 'globals'
    option ula_prefix 'fd93:f7f5:2744::/48'

config device
    option name 'br-lan'
    option type 'bridge'
    list ports 'lan1'
    list ports 'lan2'
    list ports 'lan3'
    list ports 'lan4'

config interface 'lan'
    option device 'br-lan'
    option proto 'static'
    option ipaddr '192.168.1.1'
    option netmask '255.255.255.0'
    option ip6assign '60'

config interface 'wan'
    option device 'eth0.2'
    option proto 'dhcp'

config interface 'wan6'
    option device 'eth0.2'
    option proto 'dhcpv6'

config interface 'wwan'
    option proto 'dhcp'
    option device 'phy1-sta0'

config switch
    option name 'switch0'
    option reset '1'
    option enable_vlan '1'

config switch_vlan
    option device 'switch0'
    option vlan '1'
    option ports '1 2 3 4 6t'
    option vid '1'

config switch_vlan
    option device 'switch0'
    option vlan '2'
    option ports '0 6t'
    option vid '2'
```

### Wireless (UCI)
```bash
# /etc/config/wireless
config wifi-device 'radio0'
    option type 'mac80211'
    option path 'platform/10300000.wmac'
    option channel '1'
    option band '2g'
    option htmode 'HT20'
    option country 'IN'
    option cell_density '0'

config wifi-iface 'default_radio0'
    option device 'radio0'
    option network 'lan'
    option mode 'ap'
    option ssid 'HomeLab-2G'
    option encryption 'psk2+ccmp'
    option key 'lundlemera'
    option disabled '0'

config wifi-device 'radio1'
    option type 'mac80211'
    option path 'pci0000:00/0000:00:00.0/0000:01:00.0'
    option channel '149'
    option band '5g'
    option htmode 'VHT40'
    option country 'IN'
    option cell_density '0'

config wifi-iface 'wwan'
    option device 'radio1'
    option network 'wwan'
    option mode 'sta'
    option ssid 'Rishihood_Learner'
    option encryption 'psk2'
    option key 'HomeLab5G-SecurePass2026!'
    option disabled '0'

config wifi-iface 'default_radio1'
    option device 'radio1'
    option network 'lan'
    option mode 'ap'
    option ssid 'HomeLab-5G'
    option encryption 'sae-mixed'
    option key 'HomeLab5G-SecurePass2026!'
    option disabled '1'
```

### DHCP & DNS (UCI)
```bash
# /etc/config/dhcp
config dnsmasq
    option domain 'lan'
    option local '/lan/'
    option expandhosts '1'
    option authoritative '1'
    option readethers '1'
    option leasefile '/tmp/dhcp.leases'
    option resolvfile '/tmp/resolv.conf.d/resolv.conf.auto'
    option nonegcache '0'
    option cachesize '1000'
    option localise_queries '1'
    option rebind_protection '1'
    option rebind_localhost '1'
    option filterwin2k '0'
    option localservice '1'
    option nonwildcard '1'
    option dnssec '1'
    option ednspacket_max '1232'
    list server '1.1.1.1'
    list server '8.8.8.8'
    list server '9.9.9.9'

config dhcp 'lan'
    option interface 'lan'
    option start '100'
    option limit '150'
    option leasetime '12h'
    option dhcpv6 'server'
    option ra 'server'
    option ra_slaac '1'

config host
    option name 'proxmox-host'
    option mac '00:e0:4c:68:5e:a8'
    option ip '192.168.1.10'

config host
    option name 'macbook'
    option mac '34:66:91:ef:4e:2d'
    option ip '192.168.1.50'
```

### Firewall (UCI)
```bash
# /etc/config/firewall
config defaults
    option syn_flood '1'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'REJECT'

config zone
    option name 'lan'
    option input 'ACCEPT'
    option output 'ACCEPT'
    option forward 'ACCEPT'
    list network 'lan'

config zone
    option name 'wan'
    option input 'REJECT'
    option output 'ACCEPT'
    option forward 'DROP'
    option masq '1'
    option mtu_fix '1'
    list network 'wan'
    list network 'wan6'
    list network 'wwan'

config forwarding
    option src 'lan'
    option dest 'wan'

config rule
    option name 'Allow-DHCP-Renew'
    option src 'wan'
    option proto 'udp'
    option dest_port '68'
    option target 'ACCEPT'
    option family 'ipv4'

config rule
    option name 'Allow-Ping'
    option src 'wan'
    option proto 'icmp'
    option icmp_type 'echo-request'
    option target 'ACCEPT'
    option family 'ipv4'

config rule
    option name 'Allow-IGMP'
    option src 'wan'
    option proto 'igmp'
    option target 'ACCEPT'
    option family 'ipv4'

config rule
    option name 'Allow-DHCPv6'
    option src 'wan'
    option proto 'udp'
    option dest_port '546'
    option target 'ACCEPT'
    option family 'ipv6'

config rule
    option name 'Allow-ICMPv6'
    option src 'wan'
    option proto 'icmp'
    list icmp_type 'echo-request'
    list icmp_type 'echo-reply'
    list icmp_type 'destination-unreachable'
    list icmp_type 'packet-too-big'
    list icmp_type 'time-exceeded'
    list icmp_type 'bad-header'
    list icmp_type 'unknown-header-type'
    list icmp_type 'router-solicitation'
    list icmp_type 'neighbour-solicitation'
    list icmp_type 'router-advertisement'
    list icmp_type 'neighbour-advertisement'
    option limit '1000/sec'
    option target 'ACCEPT'
    option family 'ipv6'

config rule
    option name 'Allow-IPSec-ESP'
    option src 'wan'
    option proto 'esp'
    option target 'ACCEPT'

config rule
    option name 'Allow-ISAKMP'
    option src 'wan'
    option proto 'udp'
    option dest_port '500'
    option target 'ACCEPT'
```

### Dropbear SSH (UCI)
```bash
# /etc/config/dropbear
config dropbear
    option Port '22'
    option Interface 'lan'
    option PasswordAuth 'off'
    option RootPasswordAuth 'off'
    option RSAAuth 'on'
    option PubkeyAuth 'on'
```

### uhttpd (LuCI) (UCI)
```bash
# /etc/config/uhttpd
config uhttpd 'main'
    list listen_http '192.168.1.1:80'
    list listen_https '192.168.1.1:443'
    option redirect_https '1'
    option home '/www'
    option rfc1918_filter '1'
    option max_requests '3'
    option max_connections '100'
    option cert '/etc/uhttpd.crt'
    option key '/etc/uhttpd.key'
    option ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH'
    option tlsciphers 'TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256'
```

### System (UCI)
```bash
# /etc/config/system
config system
    option hostname 'main-router'
    option timezone 'IST-5:30'
    option zonename 'Asia/Kolkata'
    option ttylogin '0'
    option log_size '64'
    option urandom_seed '0'

config timeserver 'ntp'
    option enabled '1'
    option enable_server '0'
    list server '0.openwrt.pool.ntp.org'
    list server '1.openwrt.pool.ntp.org'
    list server '2.openwrt.pool.ntp.org'
    list server '3.openwrt.pool.ntp.org'
```

### Live Runtime State

#### IP Addresses
```bash
# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
br-lan           UP             192.168.1.1/24 fd93:f7f5:2744::1/60 fe80::e848:b8ff:fe14:9fef/64
eth0             UP             fe80::e848:b8ff:fe14:9fef/64
eth0.2           UP             fe80::e848:b8ff:fe14:9fef/64
phy1-sta0        UP             10.7.28.50/19 fe80::ea48:b8ff:fe14:9ff1/64
phy1-ap0         DOWN           fe80::ea48:b8ff:fe14:9ff0/64
wlan0            UP             fe80::e848:b8ff:fe14:9fef/64
```

#### Routes
```bash
# ip route
default via 10.7.0.1 dev phy1-sta0 src 10.7.28.50
10.7.0.0/19 dev phy1-sta0 scope link src 10.7.28.50
192.168.1.0/24 dev br-lan scope link src 192.168.1.1
```

#### DHCP Leases
```bash
# cat /tmp/dhcp.leases
1731400000 00:e0:4c:68:5e:a8 192.168.1.10 proxmox-host *
1731400000 34:66:91:ef:4e:2d 192.168.1.50 macbook *
```

#### Wireless Status
```bash
# iwinfo
wlan0       ESSID: "HomeLab-2G"
            Mode: Master  Channel: 1 (2.412 GHz)
            Tx-Power: 20 dBm  Link Quality: 70/70
            Bit Rate: 72.2 Mbit/s

phy1-sta0   ESSID: "Rishihood_Learner"
            Mode: Client  Channel: 149 (5.745 GHz)
            Tx-Power: 20 dBm  Link Quality: 65/70
            Bit Rate: 400 Mbit/s (VHT40)

wlan1       (not created - AP disabled)
```

#### Listening Ports
```bash
# netstat -tulpn
Proto  Local Address         State    PID/Program
tcp    192.168.1.1:22        LISTEN   dropbear
tcp    192.168.1.1:80        LISTEN   uhttpd
tcp    192.168.1.1:443       LISTEN   uhttpd
udp    0.0.0.0:67            LISTEN   dnsmasq
udp    0.0.0.0:53            LISTEN   dnsmasq
tcp    192.168.1.1:53        LISTEN   dnsmasq
udp    0.0.0.0:68            LISTEN   (dhcp client on phy1-sta0)
```

---

## Proxmox Host (Dell i3-M380)

### System
```
Hostname: pve-host
OS: Proxmox VE 9.2.2 (Debian 12 bookworm)
Kernel: 6.8.12-4-pve
CPU: Intel Core i3-M380 @ 2.53GHz (2 cores, 4 threads)
RAM: 3.6 GB total, 1.2 GB used
SSD: 238 GB (15 GB used)
HDD: 465 GB (5 GB used)
Uptime: 18h 30m
```

### Network Interfaces
```bash
# /etc/network/interfaces
auto lo
iface lo inet loopback

iface enx00e04c685ea8 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports enx00e04c685ea8
    bridge-stp off
    bridge-fd 0
```

### Live Interface State
```bash
# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
enx00e04c685ea8  UP             fe80::2e0:4cff:fe68:5ea8/64
vmbr0            UP             192.168.1.10/24 fd93:f7f5:2744::/64 fe80::2e0:4cff:fe68:5ea8/64
tailscale0       UP             100.122.224.81/32 fd7a:115c:a1e0::de28:e052/128
```

### Routes
```bash
# ip route
default via 192.168.1.1 dev vmbr0
192.168.1.0/24 dev vmbr0 proto kernel scope link src 192.168.1.10
100.64.0.0/10 dev tailscale0 scope link
```

### Firewall (pve-firewall)

#### Cluster Firewall
```bash
# /etc/pve/firewall/cluster.fw
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
policy_forward: ACCEPT

[ALIASES]
local_network 192.168.1.0/24
```

#### Host Firewall
```bash
# /etc/pve/nodes/homelab/host.fw
[RULES]
IN SSH(ACCEPT) -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 8006 -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 3128 -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 8006 -source 100.122.224.81/32
IN ACCEPT -p tcp -dport 3128 -source 100.122.224.81/32
IN ACCEPT -p tcp -dport 22 -source 100.122.224.81/32
OUT ACCEPT
```

#### IP Set (PVEFW-0-management-v4)
```bash
# ipset list PVEFW-0-management-v4
Name: PVEFW-0-management-v4
Type: hash:net
Revision: 6
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 368
References: 2
Number of entries: 1
Members:
192.168.1.0/24
```

#### Tailscale Chain (ts-input)
```bash
# iptables -L ts-input -n -v
Chain ts-input (policy ACCEPT)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  tailscale0 *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     udp  --  *       *       0.0.0.0/0            0.0.0.0/0        udp dpt:41641
    0     0 RETURN     all  --  *       *       100.115.92.0/23      0.0.0.0/0
    0     0 DROP       all  --  !tailscale0 *       100.64.0.0/10      0.0.0.0/0
```

### Listening Ports
```bash
# ss -tulpn
Proto  Local Address         Peer Address  Process
tcp    0.0.0.0:22            0.0.0.0:*     sshd
tcp    0.0.0.0:8006          0.0.0.0:*     pveproxy
tcp    0.0.0.0:3128          0.0.0.0:*     spiceproxy
tcp    100.122.224.81:42522  0.0.0.0:*     tailscaled
tcp    [fd7a:115c:a1e0::de28:e052]:36222  *:*     tailscaled
udp    0.0.0.0:41641         0.0.0.0:*     tailscaled
udp    [::]:41641            [::]:*        tailscaled
```

### Services
```bash
# systemctl status pveproxy pvedaemon ssh tailscaled
pveproxy:     active (running)
pvedaemon:    active (running)
ssh:          active (running)
tailscaled:   active (running)
rpcbind:      masked (inactive)
```

### Storage
```bash
# pvesm status
Name         Type     Status  Total      Used     Available  %
local        dir      active  233G       15G      206G       6%
local-lvm    lvmthin  active  214G       0B       214G       0%
hdd-storage  dir      active  458G       5G       443G       1%
```

### Tailscale
```bash
# tailscale status
100.122.224.81  homelab          linux   active; direct; tx 1.2M rx 2.1M
100.110.79.92   antiks-macbook-pro  macOS  active; relay "bom"; tx 500K rx 800K
100.123.25.117  ipad-gen-8          iOS    offline
100.101.205.84  realme-c31          android offline
```

---

## Tailscale Network

### Mesh Topology
```
Control Machine (MacBook)     Proxmox Host
     100.110.79.92      ←DERP→     100.122.224.81
          │                            │
          │        Tailscale Mesh      │
          │     (100.64.0.0/10)        │
          ▼                            ▼
     [Offline: iPad, Phone]      [Exit Node: None]
```

### DERP Relay
```bash
# tailscale netcheck
Report:
  UDP: true
  IPv4: true
  IPv6: true
  MappingVariesByDestIP: false
  PreferredDERP: 12 (bom, Mumbai)
  DERPLatency:
    bom: 78ms
    blr: 82ms
    del: 95ms
```

---

## Connectivity Matrix (Verified)

| Source | Destination | SSH | HTTPS | Ping | Notes |
|--------|-------------|-----|-------|------|-------|
| MacBook (LAN) | Proxmox (1.10) | ✅ | ✅ | ✅ | Direct LAN |
| MacBook (TS) | Proxmox (100.122.224.81) | ✅ | ✅ | ✅ | Tailscale |
| MacBook (LAN) | OpenWrt (1.1) | ✅ | ✅ | ✅ | Direct LAN |
| MacBook (TS) | OpenWrt (1.1) | ✅ | ✅ | ✅ | ProxyJump via Proxmox |
| Proxmox | OpenWrt (1.1) | ✅ | N/A | ✅ | Direct LAN |
| OpenWrt | Internet | N/A | N/A | ✅ | Via WWAN |
| Proxmox | Internet | N/A | N/A | ✅ | Via OpenWrt NAT |

---

## Certificate Fingerprints

### OpenWrt LuCI (uhttpd)
```bash
# openssl x509 -in /etc/uhttpd.crt -noout -fingerprint -sha256
SHA256 Fingerprint=AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99
Validity: Nov 12 2026 - Dec 14 2027 (397 days)
Subject: CN=OpenWrt
```

### Proxmox Web UI (pveproxy)
```bash
# openssl x509 -in /etc/pve/local/pve-ssl.pem -noout -fingerprint -sha256
SHA256 Fingerprint=11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF
Validity: Nov 12 2026 - Nov 10 2027 (363 days)
Subject: CN=pve-host
```

---

*This document reflects live configuration as of 2026-11-12. Regenerate after any changes.*