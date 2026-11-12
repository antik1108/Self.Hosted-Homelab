# Infrastructure Audit Report
## Self.Hosted-Homelab - OpenWrt + Proxmox
**Date:** 2026-11-12  
**Auditor:** Kilo CLI  

---

## Executive Summary

This audit covers the complete homelab infrastructure consisting of:
- **OpenWrt Router:** TP-Link Archer C20 v5 (OpenWrt 25.12.6, kernel 6.12.112)
- **Proxmox VE Host:** Dell Inspiron N4010 (Proxmox VE 9.2.2, kernel 7.0.2-6-pve)
- **Control Machine:** macOS with Ansible 2.21.6

**Critical Finding:** The OpenWrt router has **severely constrained storage** (95% overlay usage, only 52KB free), preventing package installation and Ansible module file transfers. The router also **lacks opkg/package manager**, making software installation impossible without firmware rebuild.

---

## 1. SSH Connectivity Architecture

### Verified Connection Path
```
MacBook (192.168.29.x) 
    → Tailscale (100.122.224.81) 
    → Proxmox (root@100.122.224.81, password auth)
    → LAN SSH (root@192.168.1.1, password auth)
    → OpenWrt Router
```

### SSH Configuration
- **Proxmox Tailscale IP:** 100.122.224.81 (hostname: `homelab`)
- **Proxmox LAN IP:** 192.168.1.10 (vmbr0)
- **OpenWrt LAN IP:** 192.168.1.1 (br-lan)
- **Authentication:** Password-based (no SSH keys configured on either device)
- **MacBook SSH Keys:** id_ed25519 (itsantikmondal@gmail.com), id_rsa (antikmondal@Antiks-MacBook-Pro.local)

### Ansible Connectivity Status
| Target | Connection | Status |
|--------|------------|--------|
| Proxmox | Direct SSH (Tailscale) | ✅ Working |
| OpenWrt | SSH via Proxmox ProxyJump | ✅ Working (via custom wrapper) |

---

## 2. OpenWrt Router Audit (TP-Link Archer C20 v5)

### 2.1 System Identity & Firmware
| Property | Value |
|----------|-------|
| Model | TP-Link Archer C20 v5 |
| Board | tplink,archer-c20-v5 |
| CPU | MediaTek MT7628AN ver:1 eco:2 |
| OpenWrt Version | 25.12.6 (r33311-981c978eff) |
| Kernel | 6.12.112 |
| Target | ramips/mt76x8 |
| Rootfs Type | squashfs |
| Build Date | 2026-11-10 (1792273538) |
| Hostname | OpenWrt |

### 2.2 Hardware & Memory
| Resource | Value |
|----------|-------|
| Total RAM | 57,980 KB (~56.6 MB) |
| Used RAM | 22,412 KB |
| Free RAM | 16,876 KB |
| Swap | None |
| Flash (overlay) | 960 KB |
| Overlay Used | 908 KB (95%) |
| Overlay Free | 52 KB |

### 2.3 Storage Analysis (CRITICAL)
```
Filesystem          Size    Used  Available Use%
/dev/mtdblock5     960.0K  908.0K    52.0K   95%  /overlay
overlayfs:/overlay 960.0K  908.0K    52.0K   95%  /
```

**Overlay Breakdown:**
| Directory | Size |
|-----------|------|
| /overlay/upper/www | 1008.0K (LuCI static resources) |
| /overlay/upper/usr | 557.5K |
| /overlay/upper/etc | 29.0K |
| /overlay/upper/lib | 13.0K |
| /overlay/upper/run | 512B |

**Issues:**
- **95% overlay usage** - only 52 KB free
- **No opkg/package manager** - `/usr/bin/opkg` not found, no `/etc/opkg/distfeeds.conf`
- **Cannot install packages** - no space for package installation or Ansible module file transfers
- **LuCI resources consuming ~1MB** in overlay (compressed to ~1MB on disk)

### 2.4 Network Interfaces
| Interface | Device | Proto | IP Address | Status |
|-----------|--------|-------|------------|--------|
| lan | br-lan | static | 192.168.1.1/24 | UP |
| lan | br-lan | ipv6 | fd93:f7f5:2744::1/60 | UP (ULA) |
| wan | eth0.2 | dhcp | - | DOWN (pending) |
| wan6 | eth0.2 | dhcpv6 | - | DOWN (pending) |
| wwan | phy1-sta0 | dhcp | 10.7.28.50/19 | UP (Internet) |

**Switch Configuration (mt7530):**
- VLAN 1 (LAN): ports 1,2,3,4,6t (tagged CPU)
- VLAN 2 (WAN): ports 0,6t (tagged CPU)

### 2.5 Internet Connectivity (WWAN)
- **Upstream SSID:** Rishihood_Learner
- **Upstream Channel:** 149 (5745 MHz, 40 MHz width)
- **DHCP Address:** 10.7.28.50/19
- **Gateway:** 10.7.0.1
- **DHCP Server:** 10.1.0.2
- **DNS Servers:** 8.8.8.8, 10.5.7.1
- **Lease Time:** 64800s (18 hours)

### 2.6 Wireless Configuration

#### 2.4 GHz (radio0 - MT7628AN)
| Setting | Value |
|---------|-------|
| Phy | phy0 (platform/10300000.wmac) |
| Channel | 1 (2412 MHz) |
| HT Mode | HT20 (20 MHz) |
| TX Power | 20 dBm |
| SSID | OpenWrt |
| Encryption | WPA2-PSK (psk2) |
| Key | lundlemera |
| Interface | phy0-ap0 (master br-lan) |
| Status | UP, Broadcasting |

#### 5 GHz (radio1 - MT76x2 PCIe)
| Setting | Value |
|---------|-------|
| Phy | phy1 (pci0000:00/0000:00:00.0/0000:01:00.0) |
| AP Channel | 60 (5300 MHz, DFS) |
| AP HT Mode | VHT40 (40 MHz) |
| AP TX Power | 10 dBm |
| AP SSID | OpenWrt |
| AP Encryption | **NONE (OPEN!)** |
| AP Status | **DOWN** (not broadcasting) |
| STA Channel | 149 (5745 MHz, 40 MHz) |
| STA SSID | Rishihood_Learner |
| STA Encryption | WPA2-PSK |
| STA Key | sru@2021 |
| STA Interface | phy1-sta0 (UP, connected) |

**Interface Combination Limits (phy1):**
- #{ managed, AP, mesh point } ≤ 8, total ≤ 8, #channels ≤ 1
- **STA/AP BI must match** - beacon intervals must match for simultaneous operation
- Current: STA on ch 149, AP configured on ch 60 → **CHANNEL CONFLICT**

**Regulatory Domain:** `00` (DFS-UNSET) - **NOT SET FOR INDIA**

### 2.7 DHCP Configuration
| Setting | Value |
|---------|-------|
| LAN Range | 192.168.1.100 - 192.168.1.250 (150 addresses) |
| Lease Time | 12h |
| IPv6 | SLAAC + DHCPv6 (managed-config, other-config) |
| Static Reservations | None configured |
| WAN DHCP | Ignored (security) |

### 2.8 DNS Configuration (dnsmasq)
| Setting | Value |
|---------|-------|
| Upstream DNS | 1.1.1.1, 8.8.8.8, 9.9.9.9 (from config) |
| Actual WWAN DNS | 8.8.8.8, 10.5.7.1 |
| Local Domain | lan |
| DNSSEC | Enabled |
| Rebind Protection | Enabled |
| Cache Size | 1000 |
| Local Service Only | Yes |

### 2.9 Firewall Configuration
**Defaults:** input=REJECT, output=ACCEPT, forward=REJECT, synflood=1

**Zones:**
| Zone | Input | Output | Forward | Masq | Networks |
|------|-------|--------|---------|------|----------|
| lan | ACCEPT | ACCEPT | ACCEPT | - | lan |
| wan | REJECT | ACCEPT | DROP | YES | wan, wan6, wwan |

**Forwarding:** lan → wan (ACCEPT)

**Key Rules:**
- Allow-DHCP-Renew (wan UDP 68)
- Allow-Ping (wan ICMP echo-request)
- Allow-IGMP (wan)
- Allow-DHCPv6 (wan UDP 546)
- Allow-MLD (wan IPv6)
- Allow-ICMPv6-Input/Forward (wan, rate limited)
- Allow-IPSec-ESP (wan→lan)
- Allow-ISAKMP (wan→lan UDP 500)

### 2.10 SSH Configuration (dropbear)
| Setting | Value |
|---------|-------|
| Port | 22 |
| PasswordAuth | on |
| RootPasswordAuth | on |
| RootLogin | on (password) |
| BannerFile | not configured |
| Key Auth | Not explicitly configured |

### 2.11 LuCI/uhttpd Configuration
| Setting | Value |
|---------|-------|
| HTTP | 0.0.0.0:80, [::]:80 |
| HTTPS | 0.0.0.0:443, [::]:443 |
| Redirect HTTPS | Disabled |
| RFC1918 Filter | Enabled |
| Cert/Key | Self-signed (ec, P-256, 397 days) |

### 2.12 System & Time
| Setting | Value |
|---------|-------|
| Timezone | GMT0 (UTC) - **NOT Asia/Kolkata** |
| NTP Servers | 0-3.openwrt.pool.ntp.org |
| NTP Server Mode | Disabled (client only) |
| Log Size | 128 KB |

### 2.13 Listening Services
| Proto | Port | Bind Address | Process |
|-------|------|--------------|---------|
| tcp | 22 | 0.0.0.0, ::: | dropbear |
| tcp | 53 | 127.0.0.1, 10.7.28.50, 192.168.1.1, ::: | dnsmasq |
| tcp | 80 | 0.0.0.0, ::: | uhttpd |
| tcp | 443 | 0.0.0.0, ::: | uhttpd |
| udp | 53 | Multiple | dnsmasq |
| udp | 67 | 0.0.0.0 | dnsmasq (DHCP) |
| udp | 546 | ::: | odhcp6c |
| udp | 547 | ::: | odhcpd |

---

## 3. Proxmox VE Audit (Dell Inspiron N4010)

### 3.1 System Identity
| Property | Value |
|----------|-------|
| Hostname | homelab |
| Proxmox VE | 9.2.2 (pve-manager 9.2.2) |
| Kernel | 7.0.2-6-pve |
| Base OS | Debian 13 (trixie) |

### 3.2 Hardware
| Resource | Value |
|----------|-------|
| CPU | Intel Core i3-M380 @ 2.53GHz (2 cores, 4 threads) |
| Virtualization | VT-x supported |
| RAM | 3.6 GiB total, 1.7 GiB used, 1.9 GiB available |
| Swap | 4.0 GiB (254 MiB used) |

### 3.3 Storage
| Device | Size | Type | Mount/Use |
|--------|------|------|-----------|
| sda | 238.5 GB | SSD | Proxmox OS (LVM) |
| ├─ sda3 | 237.5 GB | LVM | pve VG |
| │ ├─ pve-swap | 4 GB | swap | |
| │ ├─ pve-root | 70.4 GB | ext4 | / (10% used) |
| │ └─ pve-data | 144.2 GB | LVM-thin | VM images (0% used) |
| sdb | 465.8 GB | HDD | /mnt/pve/hdd-storage (1% used) |

**PVE Storage:**
| Storage | Type | Total | Used | Available | Use% |
|---------|------|-------|------|-----------|------|
| local | dir | 72 GB | 6.8 GB | 61.5 GB | 9.4% |
| local-lvm | lvmthin | 151 GB | 0 | 151 GB | 0% |
| hdd-storage | dir | 479 GB | 2 MB | 455 GB | 0% |

### 3.4 Network
| Interface | Type | IP | Bridge Ports |
|-----------|------|-----|--------------|
| vmbr0 | static | 192.168.1.10/24 | enx00e04c685ea8 (nic0) |
| | gateway | 192.168.1.1 | |

### 3.5 VMs & Containers
**No VMs or LXC containers currently running.**

### 3.6 Security
- **SSH:** Password authentication enabled (root)
- **Firewall:** Proxmox firewall available but not configured
- **Repositories:** Enterprise repos failing (401 Unauthorized), using Debian + Tailscale repos only

---

## 4. Ansible Repository Audit

### 4.1 Structure Analysis
```
Self.Hosted-Homelab/
├── ansible.cfg          ✅ Configured (needs updates)
├── requirements.yml     ✅ Collections defined
├── inventory/
│   ├── hosts.yml        ⚠️ Needs ProxyJump config
│   └── group_vars/
│       ├── all.yml      ✅ Good separation
│       ├── openwrt.yml  ✅ Hardware-specific vars
│       └── proxmox.yml  ✅ Hardware-specific vars
├── playbooks/
│   ├── verify.yml       ✅ Read-only checks
│   ├── backup.yml       ✅ Backup configs
│   ├── openwrt.yml      ✅ Main playbook
│   └── proxmox.yml      ✅ Proxmox playbook
├── roles/
│   ├── openwrt/         ✅ Modular tasks
│   └── proxmox/         ✅ Modular tasks
├── docs/                ✅ Network design, migration guide
├── secrets/             ✅ Gitignored (empty - needs vault)
└── backups/             ✅ Gitignored
```

### 4.2 Issues Found
| Issue | Severity | Description |
|-------|----------|-------------|
| ansible.cfg callback | Medium | `community.general.yaml` callback deprecated in Ansible 2.13+ |
| Inventory ProxyJump | High | No working ProxyJump configuration for OpenWrt |
| Missing Vault | High | No Ansible Vault for secrets (WiFi passwords, SSH keys) |
| openwrt.yml vars_files | Medium | Hardcoded relative paths |
| Proxmox storage task | High | Requires `vault_proxmox_password` which doesn't exist |
| OpenWrt opkg missing | Critical | Router has no package manager, cannot install python3 |

### 4.3 Role Quality
- **openwrt role:** Well-structured, modular, uses handlers correctly
- **proxmox role:** Uses templates, but storage task needs API password
- **Handlers:** Properly defined for service reloads
- **Idempotency:** UCI modules should be idempotent, but not tested

---

## 5. Security Assessment

### Critical Vulnerabilities
| # | Issue | Impact | Recommendation |
|---|-------|--------|----------------|
| 1 | **5 GHz AP unencrypted** | HIGH | Open WiFi on radio1 (encryption=none) - currently DOWN but configured |
| 2 | **SSH password auth enabled** | HIGH | Both devices allow password auth, no key-only enforcement |
| 3 | **No SSH keys deployed** | HIGH | No authorized_keys on either device |
| 4 | **LuCI HTTP/HTTPS on all interfaces** | MEDIUM | uhttpd listening on 0.0.0.0 (including WWAN 10.7.28.50) |
| 5 | **Regulatory domain unset** | MEDIUM | DFS-UNSET, not configured for India (IN) |
| 6 | **Timezone wrong** | LOW | Router uses UTC, should be Asia/Kolkata |
| 7 | **No overlay space** | CRITICAL | Cannot install security updates or packages |

### Network Exposure
- **LuCI accessible on WWAN interface (10.7.28.50:80/443)** - exposed to upstream network
- **SSH accessible on WWAN interface (10.7.28.50:22)** - exposed to upstream network
- **DNS listening on WWAN interface** - exposes resolver to upstream

---

## 6. Identified Problems Summary

### Critical (Blockers)
1. **OpenWrt overlay 95% full (52 KB free)** - Cannot install packages, run Ansible modules requiring file transfer, or save configuration changes safely
2. **No opkg/package manager on OpenWrt** - Cannot install python3, additional tools, or security updates
3. **5 GHz AP configured without encryption** - Security risk if enabled
4. **No Ansible Vault for secrets** - WiFi passwords, SSH keys in plaintext in variables

### High Priority
5. **SSH password authentication on both devices** - Should be key-only
6. **LuCI/uhttpd listening on all interfaces (including WWAN)** - Management plane exposed upstream
7. **Regulatory domain not set for India** - Legal compliance, DFS channel issues
8. **ProxyJump not configured in Ansible** - Cannot manage OpenWrt via Ansible properly
9. **Proxmox enterprise repos failing** - No security updates from Proxmox

### Medium Priority
10. **Timezone incorrect (UTC vs Asia/Kolkata)**
11. **No static DHCP reservations configured** for Proxmox/MacBook
12. **5 GHz AP channel (60) conflicts with STA (149)** - Cannot operate simultaneously
13. **Ansible callback plugin deprecated**
14. **No backup verification process documented**

### Low Priority
15. **No IPv6 prefix delegation configured** (only ULA)
16. **No log aggregation/remote syslog**
17. **LED triggers reference old switch0 naming**
18. **Self-signed certificates for LuCI**

---

## 7. Recommendations

### Immediate Actions (Before Any Configuration Changes)
1. **Free overlay space** - Remove `/overlay/upper/www/luci-static` (regeneratable), consider removing unused packages if any
2. **Create Ansible Vault** - Encrypt WiFi passwords, SSH keys, Proxmox password
3. **Set up SSH keys** - Deploy id_ed25519.pub to both devices' authorized_keys
4. **Disable password auth** - Configure dropbear/sshd for key-only auth
5. **Fix ProxyJump in Ansible** - Use working wrapper or native ProxyJump with keys

### Short-term (Week 1)
6. **Secure LuCI** - Bind uhttpd to br-lan only, enable HTTPS redirect
7. **Set regulatory domain** - `uci set wireless.radio0.country='IN'` and `radio1.country='IN'`
8. **Fix 5 GHz AP** - Either disable or configure proper encryption + non-conflicting channel
8. **Set correct timezone** - Asia/Kolkata
9. **Add static DHCP reservations** for Proxmox (192.168.1.10) and MacBook
10. **Disable enterprise repos on Proxmox** - Configure no-subscription repo

### Medium-term (Month 1)
11. **Implement backup strategy** - Automated OpenWrt sysupgrade backups, Proxmox vzdump
12. **Configure IPv6** - Request prefix delegation from upstream if available
13. **Set up remote syslog** - Forward logs to Proxmox or external
14. **Test Ansible playbooks** - Run verify, backup, then openwrt in check mode
15. **Document recovery procedures** - For both devices

### Hardware Migration Preparation
16. **Create hardware profile template** - Document interface/radio names for future routers
17. **Test configuration on another device** - Validate portability
18. **Separate hardware-specific vars** - Already partially done in group_vars/openwrt.yml

---

## 8. Ansible Connectivity Status

### Working Configuration (for reference)
```bash
# Direct Proxmox (Tailscale)
ssh root@100.122.224.81

# OpenWrt via Proxmox (two-hop)
sshpass -p "mdantik" ssh -o ProxyCommand="sshpass -p 'tamannahazari' ssh -W %h:%p root@100.122.224.81" root@192.168.1.1

# Ansible wrapper (working)
/tmp/ansible_ssh_wrapper.sh [ssh_args] 192.168.1.1 [command]
```

### Required for Production Ansible
- SSH keys deployed to both devices
- Ansible Vault with all secrets
- ProxyJump configured in inventory or SSH config
- Overlay space freed on OpenWrt for module transfers

---

## 9. Next Steps

1. **Review this audit** with stakeholder
2. **Prioritize critical fixes** (overlay space, security)
3. **Create Ansible Vault** and populate secrets
4. **Deploy SSH keys** to both devices
5. **Fix Ansible inventory** with working ProxyJump
6. **Run backup playbook** before any changes
7. **Test verify playbook** in check mode
8. **Apply configuration changes** in small batches with verification

---

*End of Audit Report*