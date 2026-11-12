# Router Replacement Migration Guide

## Overview

This document provides a step-by-step procedure for migrating from the current TP-Link Archer C20 v5 to a new OpenWrt-supported router with dual 5GHz radios (or 6GHz) to enable simultaneous 5GHz STA (WWAN) + 5GHz AP operation.

---

## Hardware Requirements

### Target Router Specifications (Minimum)
| Requirement | Specification | Rationale |
|-------------|---------------|-----------|
| **Architecture** | MediaTek Filogic 820/830 or Qualcomm IPQ807x | OpenWrt 23.05+ support, dual 5GHz |
| **RAM** | ≥ 512 MB (1 GB preferred) | Package headroom, future services |
| **Flash/Storage** | ≥ 128 MB (8 GB eMMC preferred) | Overlay space, logs, packages |
| **5GHz Radios** | **Two independent 5GHz radios** (or 6GHz) | STA on ch149 + AP on ch36-165 simultaneously |
| **Ethernet** | 2× GbE minimum (2.5G preferred) | WAN + LAN, future-proof |
| **USB** | 1× USB 3.0 | Storage, LTE backup |
| **OpenWrt Support** | 23.05+ stable, snapshot current | Long-term support |

### Recommended Models (2026)
| Model | SoC | RAM | Storage | 5GHz Radios | Price | Notes |
|-------|-----|-----|---------|-------------|-------|-------|
| **GL.iNet GL-MT6000 (Flint 2)** | MT7986B (Filogic 820) | 1 GB | 8 GB eMMC | 2× 5GHz (4×4 + 2×2) | ~$100 | Best value, OpenWrt official |
| **Banana Pi BPI-R3** | MT7986A (Filogic 830) | 2 GB | 8 GB eMMC | 2× 5GHz (4×4 + 2×2) | ~$120 | Developer board, 4× GbE + SFP |
| **NanoPi R5S** | RK3568 | 4 GB | 16 GB eMMC | 1× 5GHz (needs USB WiFi) | ~$90 | x86-like, no built-in WiFi |
| **ASUS RT-AX86U Pro** | BCM4912 | 1 GB | 256 MB | 2× 5GHz | ~$200 | Broadcom, limited OpenWrt |
| **Xiaomi AX3000T** | MT7981B | 256 MB | 128 MB | 2× 5GHz | ~$60 | Budget, less RAM |

> **Primary Recommendation**: **GL.iNet GL-MT6000 (Flint 2)** - Official OpenWrt support, dual 5GHz, 1GB RAM, 8GB storage, good price.

---

## Pre-Migration Checklist

### 1. Backup Current Configuration
```bash
# Run full backup
ansible-playbook playbooks/backup.yml --vault-password-file secrets/vault_pass.txt

# Verify backups
ls -la backups/proxmox/$(date +%Y-%m-%d)/
ls -la backups/openwrt/$(date +%Y-%m-%d)/

# Export current configs for reference
ssh openwrt "uci show" > openwrt-current-config-$(date +%Y%m%d).txt
ssh pve "cat /etc/network/interfaces" > proxmox-network-$(date +%Y%m%d).conf
```

### 2. Document Current State
- [ ] Current IP scheme (192.168.1.0/24)
- [ ] VLAN configuration (VLAN 1 LAN, VLAN 2 WAN)
- [ ] WiFi SSIDs/passwords (2G, 5G STA, 5G AP)
- [ ] DHCP static reservations
- [ ] Firewall rules
- [ ] Port forwards (if any)
- [ ] Certificate fingerprints
- [ ] SSH keys (dropbear authorized_keys)

### 3. Prepare New Router
- [ ] Purchase target router
- [ ] Download OpenWrt firmware (stable 23.05.x or snapshot)
- [ ] Prepare TFTP/serial recovery method
- [ ] Label new router MAC addresses (LAN/WAN)

---

## Migration Procedure

### Phase 1: Stage New Router (Offline)

#### 1.1 Flash OpenWrt
```bash
# GL-MT6000: Use U-Boot web recovery or TFTP
# 1. Set PC static IP: 192.168.1.100/24
# 2. Connect to LAN port 1
# 3. Power off, hold reset, power on, wait for LED pattern
# 4. Access http://192.168.1.1 → Upload firmware
# 5. Wait for reboot

# Verify
ssh root@192.168.1.1
# Should show OpenWrt banner
```

#### 1.2 Initial Configuration (Minimal)
```bash
# Set root password
passwd

# Set hostname
uci set system.@system[0].hostname='main-router-new'
uci commit system

# Configure LAN (same subnet)
uci set network.lan.ipaddr='192.168.1.1'
uci set network.lan.netmask='255.255.255.0'
uci commit network

# Enable SSH temporarily for Ansible
uci set dropbear.@dropbear[0].Interface='lan'
uci set dropbear.@dropbear[0].PasswordAuth='on'
uci commit dropbear
/etc/init.d/dropbear restart
```

#### 1.3 Deploy SSH Key
```bash
# From control machine
cat ~/.ssh/id_rsa.pub | ssh root@192.168.1.1 "mkdir -p /etc/dropbear && cat >> /etc/dropbear/authorized_keys"
```

---

### Phase 2: Configure New Router (Parallel)

#### 2.1 Update Ansible Inventory
```yaml
# inventory/hosts.yml - Add new router alongside old
openwrt:
  hosts:
    main-router:           # OLD - keep for fallback
      ansible_host: 192.168.1.1
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      ansible_ssh_common_args: >-
        -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/id_ed25519
        -o StrictHostKeyChecking=no
        -o UserKnownHostsFile=/dev/null
        root@100.122.224.81"
    main-router-new:       # NEW - configure in parallel
      ansible_host: 192.168.1.1
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      ansible_ssh_common_args: >-
        -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/id_ed25519
        -o StrictHostKeyChecking=no
        -o UserKnownHostsFile=/dev/null
        root@100.122.224.81"
```

#### 2.2 Create New Group Vars (or update existing)
```yaml
# inventory/group_vars/openwrt.yml - Update for new hardware
router:
  model: "GL.iNet GL-MT6000"
  arch: "mediatek/filogic"
  management_interface: "br-lan"
  wan_interface: "wan"
  lan_interface: "lan"

wifi:
  radio0:                    # 2.4 GHz
    name: "radio0"
    phy: "phy0"
    band: "2g"
    channel: "1"
    htmode: "HT20"
    ssid: "HomeLab-2G"
    encryption: "psk2+ccmp"
    country: "IN"
    disabled: false

  radio1:                    # 5GHz Radio 1 - WWAN STA
    name: "radio1"
    phy: "phy1"
    band: "5g"
    channel: "149"
    htmode: "VHT80"
    ssid: "Rishihood_Learner"
    encryption: "psk2"
    country: "IN"
    disabled: false

  radio2:                    # 5GHz Radio 2 - AP (NEW!)
    name: "radio2"
    phy: "phy2"
    band: "5g"
    channel: "36"            # Different from STA channel 149
    htmode: "VHT80"
    ssid: "HomeLab-5G"
    encryption: "sae-mixed"  # WPA3
    country: "IN"
    disabled: false
```

#### 2.3 Run Ansible Configuration
```bash
# Test on new router (different host pattern)
ansible-playbook playbooks/openwrt.yml --limit main-router-new --check --diff --vault-password-file secrets/vault_pass.txt

# Apply
ansible-playbook playbooks/openwrt.yml --limit main-router-new --vault-password-file secrets/vault_pass.txt
```

---

### Phase 3: Validation (New Router Isolated)

#### 3.1 Test WWAN Connection
```bash
# Connect to college Wi-Fi
ssh main-router-new "iwinfo phy1-sta0 info"
# Should show associated with Rishihood_Learner on ch149

# Verify internet
ssh main-router-new "ping -c 3 1.1.1.1"
```

#### 3.2 Test 5GHz AP (Radio 2)
```bash
# Verify AP is up on different channel
ssh main-router-new "iwinfo wlan2 info"
# Should show Master mode, channel 36, SSID HomeLab-5G

# Connect test device (phone/laptop) to HomeLab-5G
# Verify IP from DHCP (192.168.1.100-250)
```

#### 3.3 Test LAN Services
```bash
# DHCP
ssh main-router-new "cat /tmp/dhcp.leases"

# DNS
ssh main-router-new "nslookup pve-host.lan 127.0.0.1"

# LuCI
curl -k https://192.168.1.1

# SSH
ssh main-router-new "echo OK"
```

#### 3.4 Test Proxmox Connectivity
```bash
# From Proxmox (still connected to OLD router)
ssh pve "ping -c 3 192.168.1.1"
ssh pve "ssh root@192.168.1.1 'echo OK'"  # Should reach NEW router if MAC same
```

---

### Phase 4: Cutover

#### 4.1 Schedule Maintenance Window
- Notify users (if any)
- Ensure Proxmox VMs/CTs can tolerate brief network interruption
- Have console access to both routers (serial/HDMI)

#### 4.2 Physical Swap
```
1. Power down OLD router (Archer C20)
2. Disconnect all Ethernet cables
3. Connect cables to NEW router (same ports):
   - WAN port → (unused, but connect for future)
   - LAN port 1 → Proxmox (enx00e04c685ea8)
   - LAN ports 2-4 → free
4. Power on NEW router
5. Wait 2 minutes for boot
```

#### 4.3 Verify Cutover
```bash
# From Proxmox (should work immediately if MAC same)
ssh pve "ping -c 3 192.168.1.1"
ssh pve "ssh root@192.168.1.1 'echo OK'"

# From MacBook (LAN)
ping 192.168.1.1
ssh openwrt "echo OK"  # Via ProxyJump
curl -k https://192.168.1.1

# From MacBook (Tailscale)
tailscale ping 100.122.224.81
ssh pve "ssh root@192.168.1.1 'echo OK'"
```

#### 4.4 Update MAC Addresses (if changed)
```bash
# If new router has different MAC, update:
# 1. Proxmox DHCP static reservation
# 2. OpenWrt DHCP static reservations
# 3. Proxmox /etc/network/interfaces (if using MAC-based naming)

# Via Ansible (update group_vars, re-run)
ansible-playbook playbooks/openwrt.yml --limit main-router-new --vault-password-file secrets/vault_pass.txt
ansible-playbook playbooks/proxmox.yml --vault-password-file secrets/vault_pass.txt
```

---

### Phase 5: Post-Cutover Validation

#### 5.1 Full Connectivity Test
```bash
# Run verification playbook
ansible-playbook playbooks/verify.yml --vault-password-file secrets/vault_pass.txt

# Manual checks
# - Proxmox Web UI (LAN + Tailscale)
# - Proxmox SSH (LAN + Tailscale)
# - OpenWrt LuCI (LAN + ProxyJump)
# - OpenWrt SSH (LAN + ProxyJump)
# - Internet from LAN clients
# - 5GHz AP connectivity
# - WWAN stability
```

#### 5.2 Update Documentation
```bash
# Update current-network-configuration.md with new hardware info
# Update hardware-inventory.md
# Update network-topology.md if ports changed
# Commit to git
```

#### 5.3 Decommission Old Router
```bash
# Factory reset for disposal/sale
ssh old-router "firstboot -y && reboot"
# Or keep as cold spare
```

---

## Rollback Procedure

If cutover fails:

```bash
# 1. Power down NEW router
# 2. Reconnect OLD router
# 3. Power on OLD router
# 4. Verify Proxmox connectivity
ssh pve "ping -c 3 192.168.1.1"

# 5. Troubleshoot NEW router offline
# 6. Re-attempt after fixes
```

---

## Configuration Mapping Reference

| Setting | Old (Archer C20) | New (GL-MT6000) | Notes |
|---------|------------------|-----------------|-------|
| **2.4GHz Radio** | radio0 (phy0) | radio0 (phy0) | Same config |
| **5GHz STA Radio** | radio1 (phy1) ch149 | radio1 (phy1) ch149 | Same SSID/pass |
| **5GHz AP Radio** | N/A (disabled) | **radio2 (phy2) ch36** | **NEW - different radio!** |
| **LAN Interface** | br-lan (192.168.1.1) | br-lan (192.168.1.1) | Same |
| **VLANs** | VLAN 1/2 on mt7530 | VLAN 1/2 on mt7531 | Similar, verify port map |
| **Switch Ports** | 0=WAN, 1-4=LAN | 0=WAN, 1-4=LAN | Verify `swconfig`/`dsa` |
| **Overlay** | 960 KB | ~100 MB+ | Massive improvement |

---

## Troubleshooting

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| **WWAN not connecting** | Wrong channel/password | Verify `channel='149'`, vault password |
| **5GHz AP not appearing** | Radio index wrong | Check `iw dev` → find phy2, map to radio2 |
| **VLAN not working** | DSA vs swconfig | GL-MT6000 uses DSA; check `/etc/config/network` bridge-vlan |
| **Proxmox no DHCP** | MAC changed | Update DHCP reservation, renew lease |
| **Tailscale ProxyJump fails** | OpenWrt SSH not responding | Check dropbear Interface binding, authorized_keys |

---

## Post-Migration Enhancements (New Router Capabilities)

With dual 5GHz radios and more resources:

1. **Enable 5GHz AP** - `HomeLab-5G` on radio2 (ch36, VHT80, WPA3)
2. **Install packages** - `opkg update && opkg install htop iperf3 wireguard-tools`
3. **Add monitoring** - Prometheus node_exporter, collectd
4. **WireGuard server** - For non-Tailscale VPN access
5. **VLAN expansion** - Guest WiFi (VLAN 10), IoT (VLAN 20)
6. **SQM/QoS** - `qos-scripts` or `sqm-scripts` for bufferbloat
7. **Ad blocking** - `adblock-fast` or `unbound` with blocklists

---

## Sign-off

| Phase | Date | Verified By | Status |
|-------|------|-------------|--------|
| Pre-migration backup | | | ☐ |
| New router staged | | | ☐ |
| New router configured | | | ☐ |
| Validation passed | | | ☐ |
| Cutover completed | | | ☐ |
| Post-cutover validated | | | ☐ |
| Documentation updated | | | ☐ |
| Old router decommissioned | | | ☐ |

---

*See [Hardware Inventory](../architecture/hardware-inventory.md) for device specs, [Current Network Config](../architecture/current-network-configuration.md) for live state.*