# Firewall Management

## Overview

Two independent firewalls protect this homelab:

1. **OpenWrt (fw4/nftables)** - Gateway firewall, controls WAN↔LAN traffic
2. **Proxmox (pve-firewall/iptables)** - Host firewall, controls access to Proxmox services

Additionally, **Tailscale** manages its own `ts-input` iptables chain on Proxmox.

---

## OpenWrt Firewall (fw4/nftables)

### Architecture
```
nftables tables:
├── inet fw4
│   ├── chain input_wan       # WAN → Router
│   ├── chain input_lan       # LAN → Router
│   ├── chain forward_wan     # WAN → LAN (forward)
│   ├── chain forward_lan     # LAN → WAN (forward)
│   ├── chain nat_prerouting  # DNAT
│   ├── chain nat_postrouting # SNAT/Masquerade
│   └── chain mangle_*        # QoS/marking
```

### Zone Configuration
```bash
# View current zones
ssh openwrt "fw4 print | grep -A 10 'zone'"

# Current zones:
# lan: input=ACCEPT, output=ACCEPT, forward=ACCEPT, network=lan
# wan: input=REJECT, output=ACCEPT, forward=DROP, masq=1, network="wan wan6 wwan"
```

### Key Rules

#### WAN Input (Default: REJECT)
```bash
# View WAN input rules
ssh openwrt "nft list chain inet fw4 input_wan"

# Essential allowed:
# - DHCP renew (UDP 68)
# - ICMP echo-request
# - IGMP
# - DHCPv6 (UDP 546)
# - ICMPv6 (rate limited)
# - IPsec (ESP, UDP 500)
# Everything else: REJECT
```

#### LAN Input (Default: ACCEPT)
```bash
# All LAN traffic accepted to router
# Services: SSH (22), HTTPS (443), DNS (53), DHCP (67/68), NTP (123)
```

#### Forwarding
```bash
# lan → wan: ACCEPT (internet access)
# wan → lan: DROP (no inbound from internet)
# No inter-zone forwarding except lan→wan
```

### Custom Rules

#### Add Port Forward (DNAT)
```bash
# Example: Forward WAN:8080 → LAN:192.168.1.20:80
ssh openwrt "
uci add firewall redirect
uci set firewall.@redirect[-1].name='Web-Server'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].src_dport='8080'
uci set firewall.@redirect[-1].dest='lan'
uci set firewall.@redirect[-1].dest_ip='192.168.1.20'
uci set firewall.@redirect[-1].dest_port='80'
uci set firewall.@redirect[-1].proto='tcp'
uci commit firewall
/etc/init.d/firewall reload
"
```

#### Allow Specific WAN IP
```bash
# Allow SSH from specific WAN IP (emergency only)
ssh openwrt "
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-SSH-From-Office'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].src_ip='203.0.113.50'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='22'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
/etc/init.d/firewall reload
"
```

#### Block Specific LAN Device
```bash
# Block 192.168.1.100 from internet
ssh openwrt "
uci add firewall rule
uci set firewall.@rule[-1].name='Block-Device-Internet'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].src_ip='192.168.1.100'
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].target='REJECT'
uci commit firewall
/etc/init.d/firewall reload
"
```

### View/Backup Rules
```bash
# Full nftables ruleset
ssh openwrt "nft list ruleset" > openwrt-firewall-$(date +%Y%m%d).nft

# UCI config
ssh openwrt "uci show firewall" > openwrt-firewall-uci-$(date +%Y%m%d).txt

# Human-readable
ssh openwrt "fw4 print"
```

---

## Proxmox Firewall (pve-firewall)

### Architecture
```
iptables chains (filter table):
├── INPUT
│   ├── ts-input (Tailscale chain - runs FIRST)
│   └── PVEFW-INPUT (Proxmox firewall)
├── FORWARD
│   └── PVEFW-FORWARD
├── OUTPUT
│   └── PVEFW-OUTPUT
└── Custom chains:
    ├── PVEFW-0-management-v4 (IP set)
    └── PVEFW-0-management-v6 (IP set)
```

### Configuration Files

#### Cluster Firewall (/etc/pve/firewall/cluster.fw)
```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
policy_forward: ACCEPT

[ALIASES]
local_network 192.168.1.0/24
```

#### Host Firewall (/etc/pve/nodes/homelab/host.fw)
```ini
[RULES]
# SSH from LAN
IN SSH(ACCEPT) -source 192.168.1.0/24

# Web UI from LAN
IN ACCEPT -p tcp -dport 8006 -source 192.168.1.0/24

# SPICE Proxy from LAN
IN ACCEPT -p tcp -dport 3128 -source 192.168.1.0/24

# Allow all outbound
OUT ACCEPT
```

### IP Set: PVEFW-0-management-v4
```bash
# View contents
ssh pve "ipset list PVEFW-0-management-v4"
# 192.168.1.0/24

# Default rules using this set (in cluster.fw or host.fw):
# IN SSH(ACCEPT) -source +local_network
# IN ACCEPT -p tcp -dport 8006 -source +local_network
# IN ACCEPT -p tcp -dport 3128 -source +local_network
# IN ACCEPT -p tcp -dport 5900:5999 -source +local_network
# IN ACCEPT -p tcp -dport 60000:60050 -source +local_network
```

### Tailscale Chain (ts-input)
```bash
# View
ssh pve "iptables -L ts-input -n -v"

# Rules:
# 1. ACCEPT all from tailscale0 interface
# 2. ACCEPT UDP 41641 from anywhere (DERP relay)
# 3. RETURN 100.115.92.0/23 on non-tailscale0
# 4. DROP 100.64.0.0/10 on non-tailscale0
```

### Management Commands

#### Compile & Apply
```bash
# Compile rules (check syntax)
ssh pve "pve-firewall compile"

# Start firewall
ssh pve "pve-firewall start"

# Stop firewall (emergency access)
ssh pve "pve-firewall stop"

# Status
ssh pve "pve-firewall status"

# Simulate packet (test rules)
ssh pve "pve-firewall simulate INPUT 192.168.1.50 tcp 22"
ssh pve "pve-firewall simulate INPUT 10.0.0.1 tcp 22"
ssh pve "pve-firewall simulate INPUT 100.122.224.81 tcp 8006"
```

#### View Active Rules
```bash
# All filter rules
ssh pve "iptables -L -n -v"

# Just Proxmox firewall input
ssh pve "iptables -L PVEFW-INPUT -n -v"

# Tailscale chain
ssh pve "iptables -L ts-input -n -v"
```

#### Add Custom Rule
```bash
# Edit host.fw directly
ssh pve "cat >> /etc/pve/nodes/homelab/host.fw << 'EOF'
# Custom rule
IN ACCEPT -p tcp -dport 9090 -source 192.168.1.0/24
EOF"

# Reload
ssh pve "pve-firewall compile && pve-firewall start"
```

#### Allow Tailscale IP Explicitly (Supplementary)
```bash
# Add to host.fw
ssh pve "cat >> /etc/pve/nodes/homelab/host.fw << 'EOF'
# Tailscale access (supplementary - ts-input handles primary)
IN SSH(ACCEPT) -source 100.122.224.81/32
IN ACCEPT -p tcp -dport 8006 -source 100.122.224.81/32
IN ACCEPT -p tcp -dport 3128 -source 100.122.224.81/32
EOF"

ssh pve "pve-firewall compile && pve-firewall start"
```

---

## Firewall Rule Comparison

| Traffic | OpenWrt | Proxmox | Tailscale |
|---------|---------|---------|-----------|
| LAN → OpenWrt SSH (22) | ACCEPT | N/A | N/A |
| LAN → OpenWrt HTTPS (443) | ACCEPT | N/A | N/A |
| LAN → Proxmox SSH (22) | FORWARD ACCEPT | ACCEPT (via IP set) | N/A |
| LAN → Proxmox Web UI (8006) | FORWARD ACCEPT | ACCEPT (via IP set) | N/A |
| LAN → Proxmox SPICE (3128) | FORWARD ACCEPT | ACCEPT (via IP set) | N/A |
| WAN → OpenWrt (any) | REJECT (except DHCP/ICMP) | N/A | N/A |
| WAN → Proxmox (any) | DROP (no forward) | N/A | N/A |
| Tailscale → Proxmox SSH | N/A | ACCEPT (ts-input) | ACCEPT |
| Tailscale → Proxmox Web UI | N/A | ACCEPT (ts-input) | ACCEPT |
| Internet → Tailscale UDP 41641 | N/A | ACCEPT (ts-input) | ACCEPT (DERP) |
| Proxmox → Internet | N/A | ACCEPT (policy_out) | N/A |
| OpenWrt → Internet | ACCEPT (lan→wan forward) | N/A | N/A |

---

## Testing Firewall Rules

### Proxmox Simulation
```bash
# Test from LAN (should ACCEPT)
ssh pve "pve-firewall simulate INPUT 192.168.1.50 tcp 22"
ssh pve "pve-firewall simulate INPUT 192.168.1.50 tcp 8006"

# Test from Tailscale IP (should ACCEPT via ts-input)
ssh pve "pve-firewall simulate INPUT 100.122.224.81 tcp 22"

# Test from Internet (should DROP)
ssh pve "pve-firewall simulate INPUT 10.0.0.1 tcp 22"
ssh pve "pve-firewall simulate INPUT 203.0.113.100 tcp 8006"

# Test IPv6
ssh pve "pve-firewall simulate INPUT fd93:f7f5:2744::100 tcp 22"
```

### OpenWrt Testing
```bash
# From LAN (should work)
ssh pve "ssh openwrt 'echo OK'"
curl -k https://192.168.1.1

# From WAN side (simulate - requires external host)
# Use: nmap -p 22,443 <OpenWrt-WAN-IP>

# Check connection tracking
ssh openwrt "nft list chain inet fw4 input_wan | grep -c 'counter'"
```

### Tailscale Testing
```bash
# From control machine
tailscale ping 100.122.224.81
ssh pve "echo OK"
curl -k https://100.122.224.81:8006

# Check DERP relay
tailscale netcheck
```

---

## Backup/Restore Firewall Config

### Proxmox
```bash
# Backup
ssh pve "cp /etc/pve/nodes/homelab/host.fw /etc/pve/nodes/homelab/host.fw.backup-$(date +%Y%m%d)"
ssh pve "cp /etc/pve/firewall/cluster.fw /etc/pve/firewall/cluster.fw.backup-$(date +%Y%m%d)"

# Via Ansible (in backup.yml)
ansible-playbook playbooks/backup.yml --vault-password-file secrets/vault_pass.txt

# Restore
ssh pve "cp /etc/pve/nodes/homelab/host.fw.backup-20261112 /etc/pve/nodes/homelab/host.fw"
ssh pve "pve-firewall compile && pve-firewall start"
```

### OpenWrt
```bash
# Backup UCI
ssh openwrt "uci show firewall" > backups/openwrt/firewall-uci-$(date +%Y%m%d).txt

# Backup nftables
ssh openwrt "nft list ruleset" > backups/openwrt/firewall-nft-$(date +%Y%m%d).nft

# Restore (via Ansible preferred)
ansible-playbook playbooks/openwrt.yml --tags firewall --vault-password-file secrets/vault_pass.txt

# Manual restore (nftables)
ssh openwrt "nft -f /path/to/firewall-nft-20261112.nft"
```

---

## Emergency Procedures

### Locked Out of Proxmox
```bash
# 1. From LAN (if on same network)
ssh root@192.168.1.10

# 2. From OpenWrt
ssh openwrt "ssh root@192.168.1.10"

# 3. Disable firewall temporarily
ssh pve "pve-firewall stop"

# 4. Fix rules, then re-enable
ssh pve "pve-firewall start"
```

### Locked Out of OpenWrt
```bash
# 1. From Proxmox (LAN)
ssh pve "ssh root@192.168.1.1"

# 2. Failsafe mode (physical access)
# - Connect via serial/console
# - Mount overlay: mount_root
# - Edit /etc/config/firewall, /etc/config/dropbear, /etc/config/uhttpd
# - Reboot

# 3. TFTP recovery (last resort)
```

### Tailscale Blocked
```bash
# Check ts-input chain
ssh pve "iptables -L ts-input -n -v"

# Tailscale should auto-manage; if broken:
ssh pve "systemctl restart tailscaled"

# Or reinstall
ssh pve "tailscale down && curl -fsSL https://tailscale.com/install.sh | sh && tailscale up"
```

---

## Audit Checklist

| Check | Command | Expected |
|-------|---------|----------|
| Proxmox firewall enabled | `pve-firewall status` | `enabled: 1` |
| Default policy IN | `grep policy_in /etc/pve/firewall/cluster.fw` | `DROP` |
| LAN SSH allowed | `grep 'SSH(ACCEPT)' /etc/pve/nodes/homelab/host.fw` | Present |
| LAN Web UI allowed | `grep '8006' /etc/pve/nodes/homelab/host.fw` | Present |
| Tailscale chain active | `iptables -L ts-input -n | head -5` | ACCEPT rules |
| OpenWrt WAN input REJECT | `nft list chain inet fw4 input_wan | grep reject` | Present |
| OpenWrt LAN input ACCEPT | `nft list chain inet fw4 input_lan | grep accept` | Present |
| No unwanted forwards | `nft list chain inet fw4 forward_wan | grep -v 'drop\|icmp'` | Only ICMP/IPsec |

---

*See [Daily Operations](daily-operations.md) for routine checks, [Network Management](network-management.md) for network config, [Troubleshooting](troubleshooting.md) for diagnostics.*