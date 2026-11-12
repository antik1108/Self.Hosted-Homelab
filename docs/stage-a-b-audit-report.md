# Stage A & B: Repository and Network Architecture Audit Report
**Date:** 2026-11-12  
**Status:** Discovery Complete - Ready for Review

---

## Executive Summary

This audit covers the complete homelab infrastructure:
- **MacBook** (Control Machine) → **Tailscale** → **Proxmox VE** (homelab, 100.122.224.81) → **LAN** → **OpenWrt Router** (Archer C20 v5, 192.168.1.1)
- Phase 3 backups verified and stored in `backups/`
- Ansible connectivity established for both hosts
- Critical security findings identified

---

## 1. Repository Audit (Stage A)

### Current Repository Structure
```
Self.Hosted-Homelab/
├── ansible.cfg                    # ✅ Updated for dual-host SSH
├── requirements.yml               # ✅ community.openwrt, community.general, ansible.utils, community.docker
├── inventory/
│   ├── hosts.yml                  # ✅ Updated with per-host SSH config
│   └── group_vars/
│       ├── all.yml                # ✅ Global vars (timezone, network, dhcp, firewall, ssh)
│       ├── openwrt.yml            # ✅ Hardware-specific vars (router, wifi, lan, wan, firewall_zones)
│       └── proxmox.yml            # ✅ Hardware-specific vars (bridges, storage, vm_defaults)
├── playbooks/
│   ├── verify.yml                 # ✅ Read-only verification
│   ├── backup.yml                 # ✅ Backup automation
│   ├── openwrt.yml                # ✅ Main OpenWrt playbook
│   └── proxmox.yml                # ✅ Main Proxmox playbook
├── roles/
│   ├── openwrt/                   # ✅ Modular tasks (system, network, wireless, dhcp, dns, firewall, ssh, services, commit)
│   └── proxmox/                   # ✅ Modular tasks (network, storage, node, vm_defaults, backup)
├── docs/
│   ├── configuration-audit.md     # ✅ Phase 3 audit
│   ├── phase3-completion-report.md # ✅ Phase 3 summary
│   ├── migration-guide.md         # ✅ Router migration procedure
│   └── network-design.md          # ✅ Network design document
├── secrets/
│   ├── vault.yml                  # ✅ Encrypted (Ansible Vault)
│   └── vault_pass.txt             # ✅ Vault password (600 perms)
└── backups/
    ├── openwrt/2026-11-12/        # ✅ Sysupgrade + UCI configs
    └── proxmox/2026-11-12/        # ✅ Interfaces, storage, pve config
```

### Issues Found in Repository

| File | Issue | Severity |
|------|-------|----------|
| `ansible.cfg` | `ssh_executable` wrapper was global, breaking Proxmox key auth | 🔴 Critical (Fixed) |
| `inventory/hosts.yml` | Missing per-host SSH config for ProxyJump | 🔴 Critical (Fixed) |
| `inventory/group_vars/all.yml` | `firewall.defaults.input: ACCEPT` - should be REJECT for WAN | 🟠 High |
| `inventory/group_vars/openwrt.yml` | `firewall_zones[1].network: "wan"` - missing `wan6` and `wwan` | 🟠 High |
| `inventory/group_vars/openwrt.yml` | Static DHCP MACs are placeholders (aa:bb:cc:dd:ee:ff) | 🟠 High |
| `roles/openwrt/tasks/firewall.yml` | Missing IPv6 ICMP rules for neighbor discovery | 🟡 Medium |
| `roles/openwrt/tasks/ssh.yml` | `RootPasswordAuth` set to `permit_root_login` value (string) not boolean | 🟡 Medium |
| `roles/proxmox/tasks/network.yml` | Uses template `interfaces.j2` which doesn't exist | 🟠 High |

---

## 2. Network Architecture Audit (Stage B)

### Verified Topology

```mermaid
graph TB
    subgraph "MacBook (Control)"
        MB[MacBook\n192.168.29.0/24\nTailscale: 100.110.79.92]
    end
    
    subgraph "Tailscale Overlay"
        TS[Tailscale DERP/Relay\nbom]
    end
    
    subgraph "Proxmox VE (homelab)"
        PVE[Proxmox VE 9.2.2\nDell Inspiron N4010\nIntel i3-M380, 4GB RAM\n238GB SSD + 465GB HDD]
        VMBr0[vmbr0: 192.168.1.10/24\nBridge: enx00e04c685ea8]
        TS0[tailscale0: 100.122.224.81/32]
    end
    
    subgraph "OpenWrt Router (Archer C20 v5)"
        OWRT[OpenWrt 25.12.6\nMT7628AN, 64MB RAM, 16MB Flash]
        BrLAN[br-lan: 192.168.1.1/24\nVLAN1: eth0.1 (ports 1-4)]
        WAN[eth0.2: VLAN2 (port 0)\nDHCP - DOWN]
        WWAN[phy1-sta0: 10.7.28.50/19\nSSID: Rishihood_Learner\nCH149, VHT40]
        Radio0[phy0-ap0: CH1, HT20\nSSID: OpenWrt\nWPA2-PSK]
        Radio1AP[phy1-ap0: CH60, VHT40\nSSID: OpenWrt\nNO ENCRYPTION - DOWN]
    end
    
    MB -->|Tailscale| TS
    TS -->|100.122.224.81| TS0
    TS0 --> PVE
    PVE -->|LAN| VMBr0
    VMBr0 -->|eth0.1| BrLAN
    BrLAN -->|LAN ports| OWRT
    OWRT -->|eth0.2| WAN
    OWRT -->|phy1-sta0| WWAN
    WWAN -.->|Upstream GW 10.7.0.1| Internet
    Radio0 -.->|2.4GHz Clients| BrLAN
    Radio1AP -.->|5GHz AP (DOWN)| BrLAN
```

### Proxmox Network Configuration (Verified)

| Interface | Type | IP Address | Status | Notes |
|-----------|------|------------|--------|-------|
| lo | loopback | 127.0.0.1/8, ::1/128 | UP | |
| enx00e04c685ea8 | physical | 192.168.1.196/24 (DHCP) | UP | Also has static 192.168.1.10 on vmbr0 |
| wlp3s0 | wifi | - | DOWN | Unused |
| vmbr0 | bridge | 192.168.1.10/24 (static) | UP | Bridge port: enx00e04c685ea8, Gateway: 192.168.1.1 |
| tailscale0 | p2p | 100.122.224.81/32, fd7a:115c:a1e0::de28:e052/128 | UP | Tailscale interface |

**Routing Table:**
```
default via 192.168.1.1 dev vmbr0
192.168.1.0/24 dev vmbr0 proto kernel scope link src 192.168.1.10
192.168.1.0/24 dev enx00e04c685ea8 proto kernel scope link src 192.168.1.196
```

**DNS:** `nameserver 192.168.1.1` (OpenWrt), domain `lan`, search `lan`

**Listening Services (Proxmox):**
| Port | Service | Bind | Security |
|------|---------|------|----------|
| 22 | sshd | 0.0.0.0, :: | Key + password auth |
| 8006 | pveproxy | *:* | Proxmox Web UI - **EXPOSED ON ALL INTERFACES** |
| 3128 | spiceproxy | *:* | SPICE proxy |
| 85 | pvedaemon | 127.0.0.1 | Local only |
| 25 | postfix | 127.0.0.1, ::1 | Local only |
| 111 | rpcbind | 0.0.0.0, :: | **EXPOSED** |
| 41641 | tailscaled | 0.0.0.0, :: | Tailscale |
| 323 | chronyd | 127.0.0.1, ::1 | NTP |

### OpenWrt Network Configuration (Verified)

| Interface | Device | Proto | IP Address | Status |
|-----------|--------|-------|------------|--------|
| loopback | lo | static | 127.0.0.1/8 | UP |
| lan | br-lan | static | 192.168.1.1/24 | UP |
| lan | br-lan | ipv6 | fd93:f7f5:2744::1/60 (ULA) | UP |
| wan | eth0.2 | dhcp | - | DOWN (pending) |
| wan6 | eth0.2 | dhcpv6 | - | DOWN (pending) |
| wwan | phy1-sta0 | dhcp | 10.7.28.50/19 | UP (Internet) |

**Switch Configuration (mt7530):**
- VLAN 1 (LAN): ports 1,2,3,4,6t (CPU tagged)
- VLAN 2 (WAN): ports 0,6t (CPU tagged)

**Routing Table:**
```
default via 10.7.0.1 dev phy1-sta0 src 10.7.28.50
10.7.0.0/19 dev phy1-sta0 scope link src 10.7.28.50
192.168.1.0/24 dev br-lan scope link src 192.168.1.1
fd93:f7f5:2744::/64 dev br-lan metric 1024
unreachable fd93:f7f5:2744::/48 dev lo metric 2147483647
```

**IPv6:** ULA prefix `fd93:f7f5:2744::/48` assigned, /60 delegated to lan

**DHCP (dnsmasq):**
- Range: 192.168.1.100 - 192.168.1.250 (150 addresses)
- Lease time: 12h
- IPv6: SLAAC + DHCPv6 (managed-config, other-config)
- DNS servers: 1.1.1.1, 8.8.8.8, 9.9.9.9 (config) | Actual: 8.8.8.8, 10.5.7.1 (from WWAN)
- Static reservations: NONE configured (placeholders only)

**DNS:** Listening on ALL interfaces (127.0.0.1, 192.168.1.1, 10.7.28.50, IPv6) - **EXPOSED ON WWAN**

### Wireless Configuration (Verified)

| Radio | Phy | Band | Channel | HT Mode | TX Power | SSID | Encryption | Status |
|-------|-----|------|---------|---------|----------|------|------------|--------|
| radio0 | phy0 | 2.4GHz | 1 | HT20 | 20 dBm | OpenWrt | WPA2-PSK (psk2) | UP (phy0-ap0) |
| radio1 | phy1 | 5GHz | 60 | VHT40 | 10 dBm | OpenWrt | **NONE** | DOWN (phy1-ap0) |
| radio1 | phy1 | 5GHz | 149 | VHT40 | 13 dBm | Rishihood_Learner | WPA2-PSK | UP (phy1-sta0) |

**Hardware Limitation:** Radio1 supports only **single channel** for STA+AP simultaneously (beacon interval must match). Current: STA on CH149, AP on CH60 = **CONFLICT - AP DOWN**

**Regulatory Domain:** `00` (DFS-UNSET) - **NOT SET FOR INDIA**

### Firewall Configuration (Verified)

**OpenWrt (nftables/fw4):**
```
Defaults: input=REJECT, output=ACCEPT, forward=REJECT, synflood=1

Zones:
- lan: input=ACCEPT, output=ACCEPT, forward=ACCEPT, network=lan
- wan: input=REJECT, output=ACCEPT, forward=DROP, masq=1, mtu_fix=1, network=wan wan6 wwan

Forwarding: lan → wan (ACCEPT)

WAN Input Rules (ALLOWED):
- DHCP renew (UDP 68)
- ICMP echo-request
- IGMP
- DHCPv6 (UDP 546)
- MLD (IPv6)
- ICMPv6 (rate limited 1000/sec)
- All else → REJECT

WAN Forward Rules (ALLOWED):
- ICMPv6 (rate limited)
- IPsec ESP
- ISAKMP (UDP 500)
- All else → DROP
```

**Proxmox:** No host firewall configured (pve-firewall installed but not enabled)

---

## 3. Security Audit Summary

### Critical Findings (🔴)

| # | Issue | Location | Impact |
|---|-------|----------|--------|
| 1 | **5GHz AP unencrypted** | OpenWrt wireless.radio1.default_radio1 | Open WiFi if enabled |
| 2 | **LuCI exposed on WWAN** | uhttpd listen_http/https 0.0.0.0 | Management UI on upstream network |
| 3 | **SSH exposed on WWAN** | dropbear listen 0.0.0.0:22 | Brute force from upstream |
| 4 | **DNS exposed on WWAN** | dnsmasq listen 10.7.28.50:53 | DNS amplification/open resolver |
| 5 | **Proxmox Web UI on all interfaces** | pveproxy *:8006 | Management exposed to LAN |
| 6 | **rpcbind exposed** | Proxmox 0.0.0.0:111 | RPC information leak |
| 7 | **Regulatory domain unset** | OpenWrt wireless | Illegal channels, DFS issues |
| 8 | **Password auth enabled on both** | dropbear + sshd | Brute force risk |
| 9 | **Placeholder DHCP MACs** | group_vars/openwrt.yml | Static leases won't work |
| 10 | **No static leases for Proxmox/MacBook** | OpenWrt dhcp | Unpredictable IPs |

### High Findings (🟠)

| # | Issue | Location | Impact |
|---|-------|----------|--------|
| 11 | WAN zone missing wan6/wwan in group_vars | group_vars/openwrt.yml | Config drift |
| 12 | Firewall defaults input=ACCEPT in group_vars | group_vars/all.yml | Insecure default |
| 13 | Proxmox interfaces template missing | roles/proxmox/tasks/network.yml | Playbook will fail |
| 14 | OpenWrt timezone UTC not Asia/Kolkata | system.@system[0] | Log correlation issues |
| 15 | No IPv6 WAN firewall rules for ND/RA | firewall | IPv6 connectivity issues |
| 16 | LuCI HTTP not redirected to HTTPS | uhttpd.redirect_https=0 | Credentials in cleartext |
| 17 | Self-signed cert with fake details | uhttpd.defaults | Browser warnings |
| 18 | 5GHz AP channel conflict with STA | wireless | AP cannot start |

### Medium Findings (🟡)

| # | Issue | Location | Impact |
|---|-------|----------|--------|
| 19 | IPv6 ICMP neighbor discovery not explicitly allowed | firewall | Potential IPv6 issues |
| 20 | No flow_offloading_hw (MT7628 supports) | firewall | Performance |
| 21 | SSH RootPasswordAuth = string not boolean | roles/openwrt/tasks/ssh.yml | Config error |
| 22 | No banner configured for SSH | dropbear | Compliance |
| 23 | Proxmox chronyd on 127.0.0.1 only | Proxmox | OK |
| 24 | Postfix listening on localhost | Proxmox | OK |

### Low/Informational (🟢)

| # | Issue | Location | Impact |
|---|-------|----------|--------|
| 25 | Overlay storage 93% used (72KB free) | OpenWrt | No room for packages |
| 26 | No opkg/package manager | OpenWrt | Cannot install tools |
| 27 | No remote syslog | OpenWrt/Proxmox | Log loss on reboot |
| 28 | No backup schedule automated | Both | Manual only |

---

## 4. Ansible Connectivity Status

### Verified Working
| Host | Connection Method | Auth | Status |
|------|-------------------|------|--------|
| Proxmox (pve-host) | Direct SSH (Tailscale 100.122.224.81) | ed25519 key | ✅ |
| OpenWrt (main-router) | ProxyJump via Proxmox | RSA key (dropbear limitation) | ✅ (raw module) |

### OpenWrt Python Limitation
- OpenWrt **does not have python3** installed
- `ansible.builtin.ping` module FAILS (requires Python)
- `ansible.builtin.raw` module WORKS
- `community.openwrt.uci` module WORKS (uses ucode on target)

---

## 5. Backup Verification

| Backup | Location | Size | Files | Verified |
|--------|----------|------|-------|----------|
| OpenWrt sysupgrade | backups/openwrt/2026-11-12/openwrt-backup-*.tar.gz | 7.5 KB | 32 | ✅ |
| OpenWrt UCI configs | backups/openwrt/2026-11-12/uci-configs.txt | 73 KB | All UCI | ✅ |
| Proxmox interfaces | backups/proxmox/2026-11-12/interfaces | 222 B | 1 | ✅ |
| Proxmox storage.cfg | backups/proxmox/2026-11-12/storage.cfg | 216 B | 1 | ✅ |
| Proxmox PVE config | backups/proxmox/2026-11-12/pve-config-backup.tar.gz | 13 KB | 42 | ✅ |

---

## 6. Recommended Execution Plan

### Stage C: OpenWrt Security Hardening (Requires Approval)
1. **Fix wireless security**: Set 5GHz AP encryption to WPA2/WPA3, disable or fix channel conflict
2. **Bind management to LAN only**: uhttpd (LuCI) and dropbear (SSH) to br-lan interface only
3. **Set regulatory domain**: `wireless.radio0.country='IN'`, `wireless.radio1.country='IN'`
4. **Disable password auth**: Enforce key-only SSH after verifying key access
5. **Fix firewall zone config**: Add wan6, wwan to WAN zone network list
6. **Add static DHCP leases**: Real MAC addresses for Proxmox and MacBook
7. **Set timezone**: Asia/Kolkata
7. **Enable HTTPS redirect for LuCI**

### Stage D: Proxmox Network Security (Requires Approval)
1. **Bind pveproxy to vmbr0 only**: Not all interfaces
2. **Disable rpcbind on external interfaces**: Or firewall it
3. **Enable Proxmox firewall**: Basic host protection
4. **Verify Tailscale ACLs**: Restrict access

### Stage E: SSH/Remote Access Hardening (Requires Approval)
1. **Test key-only auth** for both hosts
2. **Disable password auth** after verification
3. **Document SSH config** for easy access

### Stage F: Ansible Improvements (Requires Approval)
1. **Create missing Proxmox interfaces template** (interfaces.j2)
2. **Fix group_vars discrepancies** (firewall zones, networks)
3. **Add check-mode support** to all playbooks
4. **Create hardware abstraction** for future router migration

### Stage G: Backup/Recovery (Requires Approval)
1. **Automate backup schedule** via cron/Ansible
2. **Document recovery procedures** for each failure scenario
3. **Test backup restore** in non-destructive way

### Stage H: Validation & Documentation (Requires Approval)
1. **Run full connectivity test suite**
2. **Test firewall rules** (allowed + denied)
3. **Update all documentation** with actual values
4. **Create migration readiness checklist**

---

## 7. Questions Requiring Your Input

| Decision | Options | Recommendation |
|----------|---------|----------------|
| **5GHz AP** | A) Disable entirely<br>B) Move to CH149 (same as STA)<br>C) Keep on CH60, accept DOWN | A - Simpler, STA is primary |
| **LuCI/SSH binding** | A) Bind to br-lan only<br>B) Keep 0.0.0.0, rely on firewall | A - Defense in depth |
| **Regulatory domain** | Set to IN (India) | Yes - Required |
| **Password auth** | Disable after key verification | Yes - Security |
| **Static DHCP** | Add real MACs for Proxmox/MacBook | Yes - Predictability |
| **Proxmox pveproxy** | Bind to vmbr0 (192.168.1.10) only | Yes - Reduce exposure |

---

## Next Steps

**Please review this audit and approve:**
1. Which Critical/High findings to address in Stage C (OpenWrt)
2. Whether to proceed with Proxmox hardening (Stage D)
3. Any modifications to the execution plan

**Do not proceed with any configuration changes until approved.**

---
*End of Stage A/B Audit Report*