# Router Migration Guide

## Purpose

This guide documents the process for migrating from the current TP-Link Archer C20 v5 to a new OpenWrt-supported router while preserving the network configuration.

## Prerequisites

- New router supported by OpenWrt
- OpenWrt firmware image for the new router
- Physical access to both routers
- This repository cloned on control machine
- Ansible environment configured

## Migration Steps

### 1. Prepare New Router

```bash
# 1. Flash OpenWrt on new router
# Follow device-specific instructions from OpenWrt wiki

# 2. Set initial IP (usually 192.168.1.1)
# Connect via ethernet, access LuCI or SSH

# 3. Set root password
ssh root@192.168.1.1
passwd

# 4. Enable SSH key authentication (optional but recommended)
# Copy your public key to /etc/dropbear/authorized_keys
```

### 2. Update Hardware-Specific Variables

Edit `inventory/group_vars/openwrt.yml`:

```yaml
router:
  hostname: "main-router"
  model: "New Router Model"           # ← Update this
  management_ip: "192.168.1.1"
  management_interface: "br-lan"      # ← May change
  wan_interface: "wan"                # ← May change (e.g., eth0, eth1)
  lan_interface: "lan"                # ← May change

wifi:
  radio0:                             # ← Radio names may differ
    disabled: false
    country: "IN"
    channel: "auto"
    htmode: "HT40"
    ssid: "HomeLab-2G"
    encryption: "psk2+ccmp"
    key: "{{ vault_wifi_2g_password }}"
  radio1:                             # ← May not exist or be named differently
    disabled: false
    country: "IN"
    channel: "auto"
    htmode: "VHT80"
    ssid: "HomeLab-5G"
    encryption: "sae-mixed"
    key: "{{ vault_wifi_5g_password }}"
```

### 3. Identify Hardware Differences

Run on new router to discover interface/radio names:

```bash
# List network interfaces
ip link show

# List wireless radios
iw dev

# Check UCI network config
uci show network

# Check UCI wireless config
uci show wireless
```

Common differences:
- **Switch ports**: Different numbering or naming (eth0, eth1 vs lan1-lan4)
- **Radio names**: radio0/radio1 vs phy0/phy1 vs wlan0/wlan1
- **WAN interface**: wan, eth0, eth1 depending on port layout
- **VLAN support**: Some routers need explicit VLAN configuration

### 4. Test Configuration (Dry Run)

```bash
cd Self.Hosted-Homelab
source venv/bin/activate

# Verify connectivity to new router
ansible main-router -m ping

# Run verify playbook to see current state
ansible-playbook playbooks/verify.yml --limit openwrt

# Dry run OpenWrt playbook
ansible-playbook playbooks/openwrt.yml --check --diff
```

### 5. Apply Configuration

```bash
# Apply with vault password
ansible-playbook playbooks/openwrt.yml --vault-password-file .vault_pass

# Or if using vault password prompt
ansible-playbook playbooks/openwrt.yml --ask-vault-pass
```

### 6. Verify Migration

```bash
# Run verification playbook
ansible-playbook playbooks/verify.yml --limit openwrt

# Manual checks:
# - Internet connectivity
# - DHCP leases
# - DNS resolution
# - WiFi connectivity (both bands)
# - Proxmox reachability
# - Port forwards (if any)
```

### 7. Decommission Old Router

Option A: **Repurpose as Access Point**
```bash
# On old router (Archer C20):
# 1. Disable DHCP server
# 2. Set LAN IP to 192.168.1.2 (outside DHCP pool)
# 3. Connect LAN-to-LAN to new router
# 4. Configure same WiFi SSIDs/passwords
# 5. Disable WAN port
```

Option B: **Keep as Spare**
- Store with power adapter
- Document firmware version
- Label with "Spare - OpenWrt configured"

## Hardware Compatibility Matrix

| Feature | Archer C20 v5 | New Router | Notes |
|---------|---------------|------------|-------|
| WAN Port | 1x Gigabit | ? | Check speed |
| LAN Ports | 4x Gigabit | ? | Check count |
| 2.4GHz Radio | radio0 (MT76x8) | ? | Check driver |
| 5GHz Radio | radio1 (MT76x2) | ? | Check driver |
| VLAN Support | Yes (mt7530) | ? | Check switch chip |
| USB | 1x USB 2.0 | ? | For storage/3G |
| Flash | 16MB | ? | For packages |
| RAM | 128MB | ? | For services |

## Rollback Plan

If migration fails:

1. **Reconnect old router** - Physical swap cables
2. **Verify connectivity** - All devices should work immediately
3. **Troubleshoot new router** - Check logs, interface names
4. **Update variables** - Fix any mismatched names
5. **Retry** - Run playbook again

## Post-Migration Checklist

- [ ] Internet connectivity works
- [ ] All DHCP reservations active
- [ ] DNS resolution works (local + external)
- [ ] WiFi 2.4GHz connects and passes traffic
- [ ] WiFi 5GHz connects and passes traffic
- [ ] Proxmox management accessible
- [ ] Port forwards work (if configured)
- [ ] Firewall rules active
- [ ] NTP syncing
- [ ] Logs showing no errors
- [ ] Backup new router config: `ansible-playbook playbooks/backup.yml --limit openwrt`

## Adding New Router Support to Repository

To make future migrations easier:

1. Create device-specific variable file: `inventory/group_vars/openwrt_<model>.yml`
2. Document hardware quirks in `docs/hardware-<model>.md`
3. Add any device-specific tasks to `roles/openwrt/tasks/device_<model>.yml`
4. Update this migration guide with lessons learned

## Example: Adding a New Router Model

```bash
# 1. Create hardware-specific vars
cp inventory/group_vars/openwrt.yml inventory/group_vars/openwrt_glinet_flint.yml

# 2. Edit with correct interface/radio names
# 3. Update inventory/hosts.yml to use new vars file
# 4. Test and document
```

## References

- [OpenWrt Supported Devices](https://openwrt.org/toh/start)
- [OpenWrt UCI Configuration](https://openwrt.org/docs/guide-user/base-system/uci)
- [Ansible community.openwrt Collection](https://galaxy.ansible.com/community/openwrt)