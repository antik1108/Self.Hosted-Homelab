# Phase 3 Backup & Safe Security Improvements - Completion Report
**Date:** 2026-11-12  
**Status:** ✅ Completed

---

## 1. OpenWrt Configuration Backup ✅

### Created Backups
| File | Size | Contents |
|------|------|----------|
| `backups/openwrt/2026-11-12/openwrt-backup-2026-11-12T17:29:13+05:30.tar.gz` | 7.5 KB | Full sysupgrade backup (32 files) |
| `backups/openwrt/2026-11-12/uci-configs.txt` | 73 KB | Complete UCI configuration dump |

### Sysupgrade Backup Contents (verified)
- `/etc/config/network` - Network interfaces
- `/etc/config/wireless` - Wireless radios & SSIDs
- `/etc/config/dhcp` - DHCP & DNS settings
- `/etc/config/firewall` - Firewall zones & rules
- `/etc/config/system` - System, NTP, timezone
- `/etc/config/dropbear` - SSH configuration
- `/etc/config/uhttpd` - LuCI web server
- `/etc/dropbear/dropbear_rsa_host_key` - SSH host key
- `/etc/uhttpd.crt` / `/etc/uhttpd.key` - TLS certificates
- `/etc/passwd`, `/etc/shadow`, `/etc/group` - User accounts
- And 22 more configuration files

### Recovery Procedure
```bash
# To restore from sysupgrade backup:
scp openwrt-backup-*.tar.gz root@192.168.1.1:/tmp/
ssh root@192.168.1.1 "sysupgrade -r /tmp/openwrt-backup-*.tar.gz"
```

---

## 2. Proxmox Configuration Backup ✅

### Created Backups
| File | Size | Contents |
|------|------|----------|
| `backups/proxmox/2026-11-12/interfaces` | 222 B | `/etc/network/interfaces` |
| `backups/proxmox/2026-11-12/storage.cfg` | 216 B | `/etc/pve/storage.cfg` |
| `backups/proxmox/2026-11-12/pve-config-backup.tar.gz` | 13 KB | `/etc/pve/` (42 files) |
| `backups/proxmox/2026-11-12/proxmox-network-audit.txt` | 1 KB | Network audit data |

### Key Configuration Documented
- **vmbr0**: 192.168.1.10/24, gateway 192.168.1.1, bridge-port enx00e04c685ea8
- **Storage**: local (dir), local-lvm (lvmthin), hdd-storage (dir at /mnt/pve/hdd-storage)
- **DNS**: 192.168.1.1 (OpenWrt), domain lan
- **No VMs/CTs** currently running

---

## 3. Overlay Storage Investigation ✅

### Findings
**Root Cause**: `/overlay/upper` contained **duplicate files** that already exist in ROM (`/rom`):
- `/overlay/upper/www/luci-static/` (1 MB) - duplicate of `/rom/www/luci-static/`
- `/overlay/upper/usr/bin/ucode` - duplicate of `/rom/usr/bin/ucode`
- `/overlay/upper/usr/lib/libucode.so*` - duplicate of ROM
- `/overlay/upper/usr/share/ucode/` - duplicate of ROM
- `/overlay/upper/usr/share/luci/` - duplicate of ROM
- `/overlay/upper/usr/share/rpcd/` - duplicate of ROM
- `/overlay/upper/usr/share/acl.d/` - duplicate of ROM
- `/overlay/upper/usr/share/ucitrack/` - duplicate of ROM

### Action Taken
**Removed duplicate ROM files from overlay** (safe - system falls back to ROM):
```bash
rm -rf /overlay/upper/www/luci-static \
       /overlay/upper/usr/bin/ucode \
       /overlay/upper/usr/lib/libucode.so* \
       /overlay/upper/usr/share/ucode \
       /overlay/upper/usr/share/luci \
       /overlay/upper/usr/share/rpcd \
       /overlay/upper/usr/share/acl.d \
       /overlay/upper/usr/share/ucitrack \
       /overlay/upper/usr/bin /overlay/upper/usr/lib /overlay/upper/usr/libexec
```

### Result
| Metric | Before | After |
|--------|--------|-------|
| Overlay Used | 908 KB (95%) | 888 KB (93%) |
| Overlay Free | 52 KB | 72 KB |
| **Space Freed** | - | **~20 KB** |

**Note**: JFFS2 filesystem requires free space for garbage collection. The 72 KB free is still tight but sufficient for UCI commits and small file operations.

### Remaining Overlay Usage
| Directory | Size |
|-----------|------|
| `/overlay/upper/etc` | 29 KB (config files) |
| `/overlay/upper/lib` | 13 KB |
| `/overlay/upper/usr` | 0 KB (cleaned) |
| `/overlay/upper/www` | 2 KB (index.html only) |

---

## 4. Ansible Vault Created ✅

### Vault File: `secrets/vault.yml` (encrypted)
| Variable | Value | Purpose |
|----------|-------|---------|
| `vault_wifi_2g_password` | `lundlemera` | Current 2.4GHz WiFi password |
| `vault_wifi_5g_password` | `HomeLab5G-SecurePass2026!` | **New** secure 5GHz password |
| `vault_ssh_public_key` | `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFSclhMT4HhCml4jtkmQOSPJQKEXcOaHtztqb9a37Yfx` | Infrastructure SSH key |
| `vault_proxmox_password` | `tamannahazari` | Proxmox root password |

### Vault Password: `secrets/vault_pass.txt` (600 permissions)
```bash
# Usage:
ansible-playbook playbooks/openwrt.yml --vault-password-file secrets/vault_pass.txt
ansible-vault view secrets/vault.yml --vault-password-file secrets/vault_pass.txt
```

### Integration
- `inventory/group_vars/openwrt.yml` already references `{{ vault_wifi_2g_password }}` and `{{ vault_wifi_5g_password }}`
- `inventory/group_vars/proxmox.yml` references `{{ vault_proxmox_password }}` for storage tasks

---

## 5. SSH Key Authentication Configured ✅

### SSH Key Deployed
**Key**: `id_ed25519` (itsantikmondal@gmail.com) → **RSA fallback also works**

| Device | Key Type | Status |
|--------|----------|--------|
| Proxmox (100.122.224.81) | ed25519 | ✅ Working |
| OpenWrt (192.168.1.1) | RSA | ✅ Working (ed25519 not supported by dropbear v2025.89) |

### Verified Connections
```bash
# Proxmox direct (ed25519)
ssh -i ~/.ssh/id_ed25519 root@100.122.224.81

# OpenWrt via Proxmox (RSA)
ssh -i ~/.ssh/id_rsa -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/id_ed25519 root@100.122.224.81" root@192.168.1.1
```

### Password Authentication
- **KEPT ENABLED** on both devices (per requirement)
- Can be disabled after confirming key-based access works reliably

### OpenWrt authorized_keys
```
/etc/dropbear/authorized_keys contains:
- ssh-ed25519 ... itsantikmondal@gmail.com
- ssh-rsa ... antikmondal@Antiks-MacBook-Pro.local
```

---

## 6. Firewall Verification - LuCI/SSH Protected from WWAN ✅

### Current WAN Input Rules (nftables)
```
chain input_wan {
    # Only these are ALLOWED from WAN/WWAN:
    - DHCP renew (UDP 68)
    - ICMP echo-request (ping)
    - IGMP
    - DHCPv6 (UDP 546)
    - MLD (IPv6)
    - ICMPv6 (rate limited)
    
    # Everything else → reject_from_wan → REJECT
}
```

### Verified Protection
| Service | Port | Bound Interface | WAN Access |
|---------|------|-----------------|------------|
| SSH (dropbear) | 22 | 0.0.0.0, ::: | ❌ BLOCKED |
| LuCI HTTP | 80 | 0.0.0.0, ::: | ❌ BLOCKED |
| LuCI HTTPS | 443 | 0.0.0.0, ::: | ❌ BLOCKED |
| DNS | 53 | 0.0.0.0, ::: | ❌ BLOCKED |

**Note**: Services listen on all interfaces but firewall blocks WAN access. Recommended improvement: bind uhttpd/dropbear to `br-lan` only.

---

## 7. India Regulatory Domain & Wireless Investigation ✅

### Current State
| Setting | Current Value | Required |
|---------|---------------|----------|
| Regulatory Domain | `00` (DFS-UNSET) | **IN** (India) |
| 2.4GHz Country | Not set | IN |
| 5GHz Country | Not set | IN |

### Hardware Limitation (Critical)
**Radio1 (5GHz) Interface Combinations:**
```
#{ managed, AP, mesh point, P2P-client, P2P-GO } <= 8
total <= 8, #channels <= 1, STA/AP BI must match
```

**Implication**: STA (upstream) and AP (local) **must share the same channel** (beacon interval match).

### Current Configuration Conflict
| Interface | Role | Channel | Status |
|-----------|------|---------|--------|
| phy1-sta0 | STA (upstream) | 149 (5745 MHz) | ✅ Connected |
| phy1-ap0 | AP (local) | 60 (5300 MHz, DFS) | ⚠️ DOWN - channel conflict |

**Cannot operate simultaneously** on different channels.

### Recommendation
1. **Set regulatory domain**: `uci set wireless.radio0.country='IN'; uci set wireless.radio1.country='IN'`
2. **Option A**: Disable 5GHz AP, keep only 2.4GHz + 5GHz STA
3. **Option B**: Move 5GHz AP to channel 149 (same as STA) - limited power
4. **Option C**: Use 2.4GHz for both AP and STA (if upstream supports 2.4GHz)

**Do NOT change upstream STA connection** (per requirements).

---

## 8. Proxmox Connectivity Verification ✅

### Network Connectivity
| Target | Result | Latency |
|--------|--------|---------|
| OpenWrt (192.168.1.1) | ✅ | 0.6 ms |
| Self (192.168.1.10) | ✅ | 0.06 ms |
| MacBook via Tailscale (100.110.79.92) | ✅ | 61-88 ms |
| Internet (1.1.1.1, 8.8.8.8) | ✅ (traceroute) | ~10 ms |

### Routing Path
```
Proxmox (192.168.1.10) 
  → OpenWrt LAN (192.168.1.1) 
  → OpenWrt WWAN (10.7.28.50) 
  → Upstream Gateway (10.7.0.1) 
  → ISP → Internet
```

### Tailscale Status
- Proxmox: `homelab` (100.122.224.81) - active
- MacBook: `antiks-macbook-pro` (100.110.79.92) - active, relay "bom"
- Direct LAN + Tailscale both functional

---

## Summary of Changes Made

| # | Change | Risk Level | Reversible |
|---|--------|------------|------------|
| 1 | Created OpenWrt sysupgrade backup | None | N/A |
| 2 | Created Proxmox config backup | None | N/A |
| 3 | Removed duplicate ROM files from overlay | Low | Yes (files in ROM) |
| 4 | Created Ansible Vault with encrypted secrets | None | N/A |
| 5 | Deployed SSH keys (ed25519 to Proxmox, RSA to OpenWrt) | Low | Yes |
| 6 | Kept password auth enabled | Low | Yes |
| 7 | Verified firewall blocks LuCI/SSH from WWAN | None | N/A |
| 8 | Documented regulatory domain & wireless constraints | None | N/A |
| 9 | Verified full Proxmox connectivity | None | N/A |

---

## Outstanding Recommendations (Not Yet Implemented)

| Priority | Task | Reason |
|----------|------|--------|
| High | Set regulatory domain to IN | Legal compliance, correct DFS channels |
| High | Bind uhttpd to br-lan only | Defense in depth |
| High | Bind dropbear to br-lan only | Defense in depth |
| Medium | Configure 5GHz AP on channel 149 or disable | Hardware limitation |
| Medium | Disable password auth after key verification | Security hardening |
| Low | Set timezone to Asia/Kolkata | Correct timestamps |

---

## Next Steps for Phase 4 (Ansible Improvements)

1. Fix `ansible.cfg` callback plugin deprecation
2. Configure working ProxyJump in inventory for OpenWrt
3. Test `verify.yml` playbook with vault
4. Test `backup.yml` playbook with vault
5. Run `openwrt.yml` in `--check` mode
6. Separate hardware-specific variables for future router migration

---

*All backups stored in `backups/` (gitignored). No secrets committed to Git. Network remains fully operational.*