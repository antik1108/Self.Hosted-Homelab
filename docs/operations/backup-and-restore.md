# Backup and Restore Procedures

## Backup Overview

| System | Method | Frequency | Location |
|--------|--------|-----------|----------|
| **Proxmox Config** | Ansible + manual | Weekly + before changes | `backups/proxmox/YYYY-MM-DD/` |
| **Proxmox VMs/CTs** | vzdump | Weekly | `/var/lib/vz/dump/` or PBS |
| **OpenWrt Config** | sysupgrade backup + UCI dump | Weekly + before changes | `backups/openwrt/YYYY-MM-DD/` |
| **Ansible Vault** | Manual copy | Before password change | Password manager |

---

## Proxmox Backup

### Configuration Backup (Ansible)
```bash
# Run backup playbook
ansible-playbook playbooks/backup.yml --vault-password-file secrets/vault_pass.txt

# Creates:
# backups/proxmox/2026-11-12/
#   ├── interfaces-2026-11-12T10:30:00.bak
#   ├── host.fw-2026-11-12T10:30:00.bak
#   └── cluster.fw-2026-11-12T10:30:00.bak
```

### Manual Configuration Backup
```bash
# Network interfaces
ssh pve "cat /etc/network/interfaces" > backups/proxmox/interfaces-$(date +%Y%m%d).bak

# Host firewall
ssh pve "cat /etc/pve/nodes/homelab/host.fw" > backups/proxmox/host.fw-$(date +%Y%m%d).bak

# Cluster firewall
ssh pve "cat /etc/pve/firewall/cluster.fw" > backups/proxmox/cluster.fw-$(date +%Y%m%d).bak

# Storage config
ssh pve "cat /etc/pve/storage.cfg" > backups/proxmox/storage.cfg-$(date +%Y%m%d).bak

# Tailscale config
ssh pve "cat /var/lib/tailscale/tailscaled.state" > backups/proxmox/tailscaled.state-$(date +%Y%m%d).bak 2>/dev/null || true
```

### VM/CT Backup (vzdump)
```bash
# Backup all VMs/CTs to local storage
ssh pve "vzdump --all --storage local --mode snapshot --compress zstd --notes-template 'Auto backup {{guestname}}'"

# Backup specific VM
ssh pve "vzdump 100 --storage local --mode snapshot --compress zstd"

# List backups
ssh pve "vzdump --list local"
```

### Proxmox Backup Server (PBS) - Recommended
```bash
# If PBS available, add as storage:
# Datacenter → Storage → Add → Proxmox Backup Server
# Then schedule via GUI or:
ssh pve "vzdump --all --storage pbs --mode snapshot --compress zstd"
```

---

## OpenWrt Backup

### Configuration Backup (Ansible)
```bash
# Run backup playbook
ansible-playbook playbooks/backup.yml --vault-password-file secrets/vault_pass.txt

# Creates:
# backups/openwrt/2026-11-12/
#   ├── openwrt-backup-2026-11-12T10:30:00.tar.gz
#   └── uci-configs-2026-11-12T10:30:00.txt
```

### Manual Configuration Backup
```bash
# Sysupgrade backup (includes /etc/config/, /etc/dropbear/, /etc/uhttpd.*, etc.)
ssh openwrt "sysupgrade -b /tmp/openwrt-backup-$(date -Iseconds).tar.gz"
scp openwrt:/tmp/openwrt-backup-*.tar.gz backups/openwrt/

# UCI config dump
ssh openwrt "uci show" > backups/openwrt/uci-configs-$(date +%Y%m%d).txt

# Specific configs
ssh openwrt "cat /etc/config/network" > backups/openwrt/network-$(date +%Y%m%d).conf
ssh openwrt "cat /etc/config/wireless" > backups/openwrt/wireless-$(date +%Y%m%d).conf
ssh openwrt "cat /etc/config/firewall" > backups/openwrt/firewall-$(date +%Y%m%d).conf
ssh openwrt "cat /etc/config/dhcp" > backups/openwrt/dhcp-$(date +%Y%m%d).conf
ssh openwrt "cat /etc/config/dropbear" > backups/openwrt/dropbear-$(date +%Y%m%d).conf
ssh openwrt "cat /etc/config/uhttpd" > backups/openwrt/uhttpd-$(date +%Y%m%d).conf
ssh openwrt "cat /etc/config/system" > backups/openwrt/system-$(date +%Y%m%d).conf

# SSH keys
ssh openwrt "cat /etc/dropbear/authorized_keys" > backups/openwrt/authorized_keys-$(date +%Y%m%d)

# TLS certificates
ssh openwrt "cat /etc/uhttpd.crt" > backups/openwrt/uhttpd.crt-$(date +%Y%m%d)
ssh openwrt "cat /etc/uhttpd.key" > backups/openwrt/uhttpd.key-$(date +%Y%m%d)
```

### Verify Backup Integrity
```bash
# Check sysupgrade backup
tar -tzf backups/openwrt/openwrt-backup-2026-11-12T10:30:00.tar.gz | head -20

# Check UCI dump
head -50 backups/openwrt/uci-configs-2026-11-12.txt
```

---

## Restore Procedures

### Proxmox Configuration Restore

#### Network Interfaces
```bash
# From backup
scp backups/proxmox/interfaces-20261112.bak pve:/etc/network/interfaces
ssh pve "systemctl restart networking"
```

#### Firewall
```bash
# Host firewall
scp backups/proxmox/host.fw-20261112.bak pve:/etc/pve/nodes/homelab/host.fw
ssh pve "pve-firewall compile && pve-firewall start"

# Cluster firewall
scp backups/proxmox/cluster.fw-20261112.bak pve:/etc/pve/firewall/cluster.fw
ssh pve "pve-firewall compile && pve-firewall start"
```

#### Full Proxmox Reinstall + Restore
```bash
# 1. Reinstall Proxmox VE (same version)
# 2. Set hostname, IP, root password
# 3. Restore /etc/pve from backup (if cluster backup exists)
# 4. Or manually restore:
scp backups/proxmox/interfaces-20261112.bak pve:/etc/network/interfaces
scp backups/proxmox/host.fw-20261112.bak pve:/etc/pve/nodes/homelab/host.fw
scp backups/proxmox/cluster.fw-20261112.bak pve:/etc/pve/firewall/cluster.fw

# 5. Reinstall Tailscale
ssh pve "curl -fsSL https://tailscale.com/install.sh | sh && tailscale up --authkey=KEY"

# 6. Restore SSH keys
scp ~/.ssh/id_ed25519.pub pve:/root/.ssh/authorized_keys

# 7. Reboot and verify
ssh pve "reboot"
```

#### VM/CT Restore
```bash
# List available backups
ssh pve "vzdump --list local"

# Restore VM 100 from latest backup
ssh pve "vzdump --restore /var/lib/vz/dump/vzdump-qemu-100-2026_11_12-02_00_00.vma.zst 100 --storage local-lvm"

# Restore to different VMID
ssh pve "vzdump --restore /var/lib/vz/dump/vzdump-qemu-100-2026_11_12-02_00_00.vma.zst 200 --storage local-lvm"
```

---

### OpenWrt Configuration Restore

#### Sysupgrade Restore (Full Config)
```bash
# 1. Copy backup to router
scp backups/openwrt/openwrt-backup-2026-11-12T10:30:00.tar.gz openwrt:/tmp/

# 2. Restore and reboot
ssh openwrt "sysupgrade -r /tmp/openwrt-backup-2026-11-12T10:30:00.tar.gz && reboot"

# 3. Wait for reboot, verify
ssh openwrt "uci show network.lan"
```

#### Individual Config Restore
```bash
# Network
scp backups/openwrt/network-20261112.conf openwrt:/etc/config/network

# Wireless
scp backups/openwrt/wireless-20261112.conf openwrt:/etc/config/wireless

# Firewall
scp backups/openwrt/firewall-20261112.conf openwrt:/etc/config/firewall

# DHCP/DNS
scp backups/openwrt/dhcp-20261112.conf openwrt:/etc/config/dhcp

# SSH
scp backups/openwrt/dropbear-20261112.conf openwrt:/etc/config/dropbear
scp backups/openwrt/authorized_keys-20261112 openwrt:/etc/dropbear/authorized_keys

# LuCI
scp backups/openwrt/uhttpd-20261112.conf openwrt:/etc/config/uhttpd
scp backups/openwrt/uhttpd.crt-20261112 openwrt:/etc/uhttpd.crt
scp backups/openwrt/uhttpd.key-20261112 openwrt:/etc/uhttpd.key

# System
scp backups/openwrt/system-20261112.conf openwrt:/etc/config/system

# Apply all
ssh openwrt "/etc/init.d/network reload && /etc/init.d/firewall reload && /etc/init.d/dnsmasq reload && /etc/init.d/dropbear reload && /etc/init.d/uhttpd reload && wifi reload"
```

#### Ansible Restore (Preferred)
```bash
# Re-apply entire OpenWrt config from Ansible
ansible-playbook playbooks/openwrt.yml --vault-password-file secrets/vault_pass.txt

# Or specific tags
ansible-playbook playbooks/openwrt.yml --tags network,firewall,ssh,luci --vault-password-file secrets/vault_pass.txt
```

---

## Disaster Recovery Scenarios

### Scenario 1: Proxmox Host Failed (Hardware)
1. Install Proxmox on replacement hardware
2. Restore configs from backup
3. Restore VMs from vzdump/PBS
4. Update inventory with new Tailscale IP
5. Re-authenticate Tailscale
6. Verify all services

### Scenario 2: OpenWrt Router Bricked
1. TFTP recovery flash OpenWrt
2. Set root password
3. Deploy SSH key via Proxmox ProxyJump
4. Restore via sysupgrade backup OR run Ansible playbook
5. Verify WWAN connection, LAN DHCP, firewall

### Scenario 3: Both Devices Lost
1. Provision new Proxmox → restore configs
2. Provision new OpenWrt → restore configs
3. Re-establish Tailscale mesh
4. Verify connectivity end-to-end

### Scenario 4: Vault Password Lost
1. Vault cannot be decrypted without password
2. **Prevention**: Store vault password in password manager (1Password, Bitwarden, etc.)
3. **Recovery**: Recreate vault from known secrets (WiFi passwords, SSH pubkey, Proxmox password)

---

## Backup Retention Policy

| Backup Type | Retention | Storage |
|-------------|-----------|---------|
| Proxmox config (Ansible) | 12 months | Local + Git |
| Proxmox VM/CT (vzdump) | 4 weeks | Local / PBS |
| OpenWrt sysupgrade | 12 months | Local + Git |
| OpenWrt UCI dumps | 12 months | Local + Git |
| Vault password | Indefinite | Password manager |

### Cleanup Old Backups
```bash
# Proxmox vzdump (keep 4 weeks)
ssh pve "find /var/lib/vz/dump -name 'vzdump-*' -mtime +28 -delete"

# Local backup directories (keep 12 months)
find backups/proxmox -type d -mtime +365 -exec rm -rf {} \;
find backups/openwrt -type d -mtime +365 -exec rm -rf {} \;
```

---

## Verification Checklist

### After Backup
- [ ] Backup files exist in expected locations
- [ ] Sysupgrade backup extracts cleanly (`tar -tzf`)
- [ ] UCI dump contains expected configs
- [ ] VM backups listed in `vzdump --list`
- [ ] File sizes reasonable (not zero)

### After Restore (Test Quarterly)
- [ ] Proxmox network config applied, SSH accessible
- [ ] Proxmox firewall active, rules correct
- [ ] Tailscale connected
- [ ] OpenWrt LAN accessible, LuCI loads
- [ ] OpenWrt WWAN connected
- [ ] OpenWrt SSH key-only auth working
- [ ] DHCP serving leases
- [ ] DNS resolving
- [ ] Proxmox reachable via Tailscale from MacBook
- [ ] OpenWrt reachable via ProxyJump

---

*See [Daily Operations](daily-operations.md) for scheduled backups, [Troubleshooting](troubleshooting.md) for restore issues.*