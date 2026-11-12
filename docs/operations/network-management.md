# Network Management

## Network Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
                              NETWORK TOPOLOGY
└─────────────────────────────────────────────────────────────────────────────┘

INTERNET
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  COLLEGE WI-FI (WWAN)                                                       │
│  SSID: Rishihood_Learner                                                    │
│  Channel: 149 (5745 MHz), VHT40                                             │
│  Security: WPA2-PSK                                                         │
│  Subnet: 10.7.0.0/19                                                        │
│  Gateway: 10.7.0.1                                                          │
│  DNS: 8.8.8.8, 10.5.7.1                                                     │
│  OpenWrt IP: 10.7.28.50 (DHCP)                                              │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ phy1-sta0 (5GHz radio1 - STA mode)
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  OPENWRT ROUTER (TP-Link Archer C20 v5)                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │   RADIO 0       │  │   RADIO 1       │  │   SWITCH        │             │
│  │   (2.4 GHz)     │  │   (5 GHz)       │  │   (mt7530)      │             │
│  │   ─────────     │  │   ─────────     │  │   ─────────     │             │
│  │   AP: HomeLab-2G│  │   STA: WWAN ◄───┤  │   VLAN 1 (LAN): │             │
│  │   CH 1, HT20    │  │   CH 149, VHT40 │  │   Ports 1-4     │             │
│  │   WPA2-CCMP     │  │   AP: DISABLED  │  │   VLAN 2 (WAN): │             │
│  │   Country: IN   │  │   (Conflict)    │  │   Port 0        │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│         │                     │                     │                       │
│         │              (No AP)  │                  │                       │
│         ▼                     ▼                     ▼                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    BR-LAN (192.168.1.1/24)                          │   │
│  │   DHCP: 192.168.1.100-250  │  DNS: 1.1.1.1, 8.8.8.8, 9.9.9.9      │   │
│  │   Static: .10 (PVE), .50 (MB)                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ Ethernet (Port 1-4)
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PROXMOX HOST (Dell i3-M380)                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  vmbr0 (192.168.1.10/24)  ◄── enx00e04c685ea8 (Physical NIC)        │   │
│  │                                                                      │   │
│  │  Services:                                                          │   │
│  │  • SSH (22)           ◄── LAN + Tailscale                          │   │
│  │  • Web UI (8006)      ◄── LAN + Tailscale                          │   │
│  │  • SPICE Proxy (3128) ◄── LAN + Tailscale                          │   │
│  │  • Tailscale (41641)  ◄── Internet (DERP relay)                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ Tailscale Overlay (WireGuard)
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  TAILSCALE MESH (100.64.0.0/10)                                             │
│  • Proxmox: 100.122.224.81 (homelab)                                        │
│  • MacBook: 100.110.79.92 (antiks-macbook-pro)                             │
│  • iPad: 100.123.25.117 (ipad-gen-8) [offline]                             │
│  • Phone: 100.101.205.84 (realme-c31) [offline]                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Interface Configuration

### OpenWrt Interfaces
| Interface | Type | IP/Config | Physical | Purpose |
|-----------|------|-----------|----------|---------|
| `lan` | Bridge | Static 192.168.1.1/24 | br-lan (VLAN 1) | LAN management |
| `wan` | Ethernet | DHCP | eth0.2 (VLAN 2) | Wired WAN (unused) |
| `wan6` | Ethernet | DHCPv6 | eth0.2 | IPv6 WAN |
| `wwan` | Wireless | DHCP | phy1-sta0 | Wi-Fi upstream |
| `loopback` | Loopback | 127.0.0.1/8, ::1/128 | lo | Local |

### Proxmox Interfaces
| Interface | Type | IP/Config | Physical | Purpose |
|-----------|------|-----------|----------|---------|
| `vmbr0` | Bridge | Static 192.168.1.10/24 | enx00e04c685ea8 | LAN management + VM bridge |
| `tailscale0` | WireGuard | 100.122.224.81/32, fd7a:115c:a1e0::de28:e052/128 | Virtual | Tailscale overlay |
| `lo` | Loopback | 127.0.0.1/8, ::1/128 | lo | Local |

---

## VLAN Configuration (OpenWrt mt7530 Switch)

```bash
# Current VLAN setup (from /etc/config/network)
config switch
    option name 'switch0'
    option reset '1'
    option enable_vlan '1'

config switch_vlan
    option device 'switch0'
    option vlan '1'
    option ports '1 2 3 4 6t'     # LAN ports 1-4 + CPU (tagged)
    option vid '1'

config switch_vlan
    option device 'switch0'
    option vlan '2'
    option ports '0 6t'           # WAN port 0 + CPU (tagged)
    option vid '2'
```

### Port Mapping (Archer C20 v5)
| Physical Port | Switch Port | VLAN | Use |
|---------------|-------------|------|-----|
| WAN (Blue) | 0 | 2 (tagged) | Ethernet WAN |
| LAN 1 | 1 | 1 (untagged) | Proxmox |
| LAN 2 | 2 | 1 (untagged) | Free |
| LAN 3 | 3 | 1 (untagged) | Free |
| LAN 4 | 4 | 1 (untagged) | Free |
| CPU (internal) | 6 | 1,2 (tagged) | Router CPU |

---

## DHCP Management

### Current Leases
```bash
# View active leases
ssh openwrt "cat /tmp/dhcp.leases"

# Output format: expiry MAC IP hostname clientID
# Example:
# 1731400000 00:e0:4c:68:5e:a8 192.168.1.10 proxmox-host *
# 1731400000 34:66:91:ef:4e:2d 192.168.1.50 macbook *
```

### Add Static Reservation
```bash
# Via UCI
ssh openwrt "
uci add dhcp host
uci set dhcp.@host[-1].name='new-device'
uci set dhcp.@host[-1].mac='aa:bb:cc:dd:ee:ff'
uci set dhcp.@host[-1].ip='192.168.1.20'
uci commit dhcp
/etc/init.d/dnsmasq reload
"

# Via Ansible (preferred)
# Add to inventory/group_vars/openwrt.yml under dhcp.static_hosts
ansible-playbook playbooks/openwrt.yml --tags dhcp --vault-password-file secrets/vault_pass.txt
```

### Modify DHCP Range
```bash
ssh openwrt "
uci set dhcp.lan.start='50'
uci set dhcp.lan.limit='200'
uci commit dhcp
/etc/init.d/dnsmasq reload
"
```

---

## DNS Management

### Local DNS (dnsmasq)
```bash
# Add local hostname
ssh openwrt "
uci add_list dhcp.@dnsmasq[0].host='/newhost.lan/192.168.1.30'
uci commit dhcp
/etc/init.d/dnsmasq reload
"

# View all local records
ssh openwrt "cat /etc/hosts && cat /tmp/hosts.dnsmasq"
```

### Upstream DNS Servers
```bash
# Current: 1.1.1.1, 8.8.8.8, 9.9.9.9
# Change:
ssh openwrt "
uci del_list dhcp.@dnsmasq[0].server='1.1.1.1'
uci add_list dhcp.@dnsmasq[0].server='1.1.1.1'
uci add_list dhcp.@dnsmasq[0].server='9.9.9.9'
uci commit dhcp
/etc/init.d/dnsmasq reload
"
```

### DNS Troubleshooting
```bash
# Test resolution from OpenWrt
ssh openwrt "nslookup google.com 1.1.1.1"
ssh openwrt "nslookup pve-host.lan 127.0.0.1"

# Test from Proxmox
ssh pve "nslookup google.com 192.168.1.1"
ssh pve "nslookup main-router.lan 192.168.1.1"

# Check dnsmasq status
ssh openwrt "logread -e dnsmasq | tail -20"
```

---

## Wireless Management

### 2.4 GHz AP (radio0)
```bash
# Status
ssh openwrt "iwinfo wlan0 info"
ssh openwrt "iwinfo wlan0 assoclist"

# Change channel
ssh openwrt "
uci set wireless.radio0.channel='6'
uci commit wireless
wifi reload
"

# Change SSID/Password
ssh openwrt "
uci set wireless.default_radio0.ssid='New-SSID'
uci set wireless.default_radio0.key='new-password'
uci commit wireless
wifi reload
"
```

### 5 GHz STA (radio1 - WWAN)
```bash
# Status
ssh openwrt "iwinfo phy1-sta0 info"
ssh openwrt "iwinfo phy1-sta0 assoclist"

# Reconnect
ssh openwrt "
wifi down radio1
sleep 2
wifi up radio1
"

# Scan for networks
ssh openwrt "iwinfo phy1 scan"
```

### 5 GHz AP (radio1 - DISABLED)
```bash
# Enable (will conflict with STA on different channel)
ssh openwrt "
uci set wireless.default_radio1.disabled='0'
# MUST match STA channel: 149
uci set wireless.radio1.channel='149'
uci commit wireless
wifi reload
"

# Check status
ssh openwrt "iwinfo wlan1 info"
```

### Regulatory Domain
```bash
# Current: IN (India)
# Check
ssh openwrt "iw reg get"

# Change (requires reboot for full effect)
ssh openwrt "
uci set wireless.radio0.country='US'
uci set wireless.radio1.country='US'
uci commit wireless
wifi reload
"
```

---

## Routing & Failover

### Current Routes (OpenWrt)
```bash
ssh openwrt "ip route show"
# default via 10.7.0.1 dev phy1-sta0 src 10.7.28.50
# 10.7.0.0/19 dev phy1-sta0 scope link src 10.7.28.50
# 192.168.1.0/24 dev br-lan scope link src 192.168.1.1
```

### Add Static Route
```bash
# Example: Route 10.10.0.0/16 via 192.168.1.5
ssh openwrt "
uci set network.route_10_10=route
uci set network.route_10_10.interface='lan'
uci set network.route_10_10.target='10.10.0.0/16'
uci set network.route_10_10.gateway='192.168.1.5'
uci commit network
/etc/init.d/network reload
"
```

### Multi-WAN (Future)
```bash
# Install mwan3
# opkg update && opkg install mwan3 luci-app-mwan3

# Configure in /etc/config/mwan3
# interface wan { ... }
# interface wwan { ... }
# policy balanced { list use wan; list use wwan; }
# rule default { policy balanced; }
```

---

## IPv6 Management

### ULA Prefix (OpenWrt)
```bash
# Current: fd93:f7f5:2744::/48
# View
ssh openwrt "uci show network.globals.ula_prefix"

# Change (generates new random /48)
ssh openwrt "
uci set network.globals.ula_prefix='$(openssl rand -hex 6 | sed 's/\(..\)/\1:/g; s/:$//')::/48'
uci commit network
/etc/init.d/network reload
"
```

### IPv6 Firewall
```bash
# Allow ICMPv6 (required for IPv6)
ssh openwrt "
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-ICMPv6-All'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='icmp'
uci set firewall.@rule[-1].icmp_type='echo-request echo-reply destination-unreachable packet-too-big time-exceeded bad-header unknown-header-type router-solicitation neighbour-solicitation router-advertisement neighbour-advertisement'
uci set firewall.@rule[-1].family='ipv6'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
/etc/init.d/firewall reload
"
```

---

## Network Diagnostics

### Ping/Traceroute
```bash
# From Proxmox
ssh pve "ping -c 3 192.168.1.1"
ssh pve "ping -c 3 10.7.0.1"
ssh pve "ping -c 3 1.1.1.1"
ssh pve "traceroute 1.1.1.1"

# From OpenWrt
ssh openwrt "ping -c 3 192.168.1.10"
ssh openwrt "ping -c 3 10.7.0.1"
ssh openwrt "traceroute 1.1.1.1"

# From MacBook (via Tailscale)
ping 100.122.224.81
traceroute 100.122.224.81
```

### Bandwidth Test
```bash
# iperf3 (install on Proxmox: apt install iperf3)
# Server on Proxmox
ssh pve "iperf3 -s -D"

# Client from MacBook
iperf3 -c 100.122.224.81

# Client from OpenWrt (if installed)
# opkg install iperf3  # No opkg - use Proxmox as server
```

### Packet Capture
```bash
# OpenWrt (tcpdump)
ssh openwrt "tcpdump -i br-lan -n -s 0 -w /tmp/capture.pcap"
scp openwrt:/tmp/capture.pcap ./capture.pcap

# Proxmox
ssh pve "tcpdump -i vmbr0 -n -s 0 -w /tmp/capture.pcap port 22 or port 8006"
scp pve:/tmp/capture.pcap ./capture.pcap

# Analyze with Wireshark locally
```

---

## Common Network Tasks

| Task | Command |
|------|---------|
| **Restart networking (OpenWrt)** | `ssh openwrt "/etc/init.d/network restart"` |
| **Restart networking (Proxmox)** | `ssh pve "systemctl restart networking"` |
| **Flush DHCP leases** | `ssh openwrt "rm /tmp/dhcp.leases && /etc/init.d/dnsmasq restart"` |
| **Renew WWAN DHCP** | `ssh openwrt "ifdown wwan && ifup wwan"` |
| **Reload firewall** | `ssh openwrt "/etc/init.d/firewall reload"` |
| **Reload DNS** | `ssh openwrt "/etc/init.d/dnsmasq reload"` |
| **Check port listening** | `ssh pve "ss -tulpn \| grep -E '22\|8006\|3128'"` |
| **View ARP table** | `ssh openwrt "ip neigh show"` |
| **View neighbor cache** | `ssh openwrt "ip -6 neigh show"` |

---

*See [Daily Operations](daily-operations.md) for routine checks, [Firewall Management](firewall-management.md) for firewall rules, [Troubleshooting](troubleshooting.md) for diagnostics.*