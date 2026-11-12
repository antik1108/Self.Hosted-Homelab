# Daily Operations

## Routine Checks

### Morning Health Check (5 minutes)
```bash
# 1. Check Tailscale connectivity
tailscale status
tailscale ping 100.122.224.81

# 2. Verify Proxmox services
ssh pve "systemctl is-active pveproxy pvedaemon tailscaled ssh"

# 3. Verify OpenWrt services
ssh openwrt "/etc/init.d/dropbear status && /etc/init.d/uhttpd status && /etc/init.d/dnsmasq status && /etc/init.d/firewall status"

# 4. Check disk space
ssh pve "df -h / /var/lib/vz /mnt/pve/hdd-storage"
ssh openwrt "df -h /overlay"

# 5. Check memory/CPU
ssh pve "free -h && uptime"
ssh openwrt "free && uptime"
```

### Weekly Deep Check (30 minutes)
```bash
# 1. Run Ansible verification
ansible-playbook playbooks/verify.yml --vault-password-file secrets/vault_pass.txt

# 2. Check for updates
ssh pve "apt update && apt list --upgradable"
ssh openwrt "opkg list-upgradable"  # Will fail - no opkg; check releases manually

# 3. Review logs
ssh pve "journalctl -p err -u pveproxy -u ssh -u tailscaled --since '1 week ago' --no-pager"
ssh openwrt "logread -e 'error\|warn' --since '1 week ago'"

# 4. Verify backups exist
ls -la backups/proxmox/$(date +%Y-%m-%d)/
ls -la backups/openwrt/$(date +%Y-%m-%d)/

# 5. Check Tailscale device status
tailscale status --json | jq '.Peer[] | {name: .HostName, ip: .TailscaleIPs[0], online: .Online, lastSeen: .LastSeen}'

# 6. Check certificate expiry
ssh pve "openssl x509 -in /etc/pve/local/pve-ssl.pem -noout -dates"
ssh openwrt "openssl x509 -in /etc/uhttpd.crt -noout -dates"
```

### Monthly Audit (1 hour)
```bash
# 1. Full backup
ansible-playbook playbooks/backup.yml --vault-password-file secrets/vault_pass.txt

# 2. Firewall rule review
ssh pve "pve-firewall compile && cat /etc/pve/nodes/homelab/host.fw"
ssh openwrt "nft list ruleset | head -100"

# 3. SSH key audit
ssh pve "cat /root/.ssh/authorized_keys"
ssh openwrt "cat /etc/dropbear/authorized_keys"

# 4. Vault accessibility test
ansible-vault view secrets/vault.yml --vault-password-file secrets/vault_pass.txt

# 5. Overlay space trend
ssh openwrt "df -h /overlay"

# 6. Firmware/security updates check
# Check OpenWrt releases: https://openwrt.org/releases
# Check Proxmox releases: https://pve.proxmox.com/wiki/Roadmap

# 7. Tailscale ACL review (admin panel)
# https://login.tailscale.com/admin/acls
```

---

## Common Operations

### Restart Services

#### Proxmox
```bash
# Web UI
ssh pve "systemctl restart pveproxy"

# Firewall
ssh pve "pve-firewall restart"

# Tailscale
ssh pve "systemctl restart tailscaled"

# Networking (CAREFUL - drops connections)
ssh pve "systemctl restart networking"

# Full reboot
ssh pve "reboot"
```

#### OpenWrt
```bash
# Network
ssh openwrt "/etc/init.d/network restart"

# Firewall
ssh openwrt "/etc/init.d/firewall restart"

# DNS/DHCP
ssh openwrt "/etc/init.d/dnsmasq restart"

# SSH
ssh openwrt "/etc/init.d/dropbear restart"

# Web UI
ssh openwrt "/etc/init.d/uhttpd restart"

# Wireless
ssh openwrt "wifi reload"

# Full reboot
ssh openwrt "reboot"
```

### Apply Configuration Changes

#### Proxmox (via Ansible)
```bash
# Check mode
ansible-playbook playbooks/proxmox.yml --check --diff --vault-password-file secrets/vault_pass.txt

# Apply
ansible-playbook playbooks/proxmox.yml --vault-password-file secrets/vault_pass.txt

# Specific role tag
ansible-playbook playbooks/proxmox.yml --tags firewall --vault-password-file secrets/vault_pass.txt
```

#### OpenWrt (via Ansible)
```bash
# Check mode
ansible-playbook playbooks/openwrt.yml --check --diff --vault-password-file secrets/vault_pass.txt

# Apply
ansible-playbook playbooks/openwrt.yml --vault-password-file secrets/vault_pass.txt

# Specific tag
ansible-playbook playbooks/openwrt.yml --tags wireless --vault-password-file secrets/vault_pass.txt
```

#### Manual UCI Changes (OpenWrt)
```bash
# Edit config
ssh openwrt "uci set wireless.radio0.channel=6"
ssh openwrt "uci commit wireless"
ssh openwrt "wifi reload"

# Or edit file directly
ssh openwrt "vi /etc/config/wireless"
ssh openwrt "/etc/init.d/network reload"
```

---

## Monitoring Commands

### Network Connectivity
```bash
# Proxmox
ssh pve "ip -br a"
ssh pve "ip route show"
ssh pve "ss -tulpn | grep -E '22|8006|3128|41641'"

# OpenWrt
ssh openwrt "ip -br a"
ssh openwrt "ip route show"
ssh openwrt "netstat -tulpn"

# Tailscale
tailscale status
tailscale ping 100.122.224.81
tailscale netcheck
```

### Firewall Status
```bash
# Proxmox
ssh pve "pve-firewall status"
ssh pve "iptables -L -n -v | head -50"
ssh pve "iptables -t filter -L PVEFW-INPUT -n -v"

# OpenWrt
ssh openwrt "nft list ruleset"
ssh openwrt "fw4 print"
```

### Resource Usage
```bash
# Proxmox
ssh pve "pvesh get /nodes/homelab/status"
ssh pve "df -h"
ssh pve "lvs"

# OpenWrt
ssh openwrt "df -h"
ssh openwrt "free"
ssh openwrt "cat /proc/meminfo"
```

### Logs
```bash
# Proxmox
ssh pve "journalctl -u pveproxy -f"
ssh pve "journalctl -u ssh -f"
ssh pve "journalctl -u tailscaled -f"
ssh pve "tail -f /var/log/pve/tasks/index"

# OpenWrt
ssh openwrt "logread -f"
ssh openwrt "logread -e 'dropbear\|uhttpd\|dnsmasq\|firewall'"
ssh openwrt "dmesg -T | tail -30"
```

---

## Emergency Procedures

### Lost Tailscale Access to Proxmox
```bash
# 1. From LAN (if physically accessible)
ssh root@192.168.1.10

# 2. From OpenWrt (if SSH works)
ssh openwrt "ssh root@192.168.1.10"

# 3. Reset Tailscale on Proxmox
ssh pve "tailscale down && tailscale up --authkey=NEW_KEY"
```

### OpenWrt Unreachable
```bash
# 1. Check Proxmox can reach LAN
ssh pve "ping -c 3 192.168.1.1"

# 2. Check OpenWrt from Proxmox
ssh pve "ssh -i /root/.ssh/id_rsa root@192.168.1.1"

# 3. Power cycle router (physical)
# 4. TFTP recovery if bricked
```

### Proxmox Web UI Down
```bash
# 1. Check service
ssh pve "systemctl status pveproxy"

# 2. Check firewall
ssh pve "pve-firewall stop"  # Test without firewall
ssh pve "pve-firewall start"

# 3. Check cert
ssh pve "openssl x509 -in /etc/pve/local/pve-ssl.pem -noout -dates"

# 4. Restart
ssh pve "systemctl restart pveproxy"
```

### Overlay Full (OpenWrt)
```bash
# Check
ssh openwrt "df -h /overlay"

# Clean ROM duplicates
ssh openwrt "
rm -rf /overlay/upper/www/luci-static \
       /overlay/upper/usr/bin/ucode \
       /overlay/upper/usr/lib/libucode.so* \
       /overlay/upper/usr/share/ucode \
       /overlay/upper/usr/share/luci \
       /overlay/upper/usr/share/rpcd \
       /overlay/upper/usr/share/acl.d \
       /overlay/upper/usr/share/ucitrack
"

# Verify
ssh openwrt "df -h /overlay"
```

---

## Scheduled Maintenance Windows

| Task | Frequency | Window | Duration |
|------|-----------|--------|----------|
| Health check | Daily | Morning | 5 min |
| Log review | Weekly | Sunday 10 AM | 30 min |
| Full backup | Weekly | Sunday 2 AM | 15 min |
| Update check | Monthly | 1st Saturday | 1 hour |
| Firmware upgrade | Quarterly | Planned | 2 hours |
| DR test | Semi-annually | Planned | 4 hours |

---

## Change Management

### Before Any Change
1. Run `ansible-playbook playbooks/backup.yml`
2. Run playbook in `--check --diff` mode
3. Document change in `CHANGELOG.md`
4. Have rollback plan (backup restore)

### After Change
1. Verify connectivity: `ansible-playbook playbooks/verify.yml`
2. Run health checks
3. Update documentation if config changed
4. Commit changes to git

---

*See [Network Management](network-management.md), [Firewall Management](firewall-management.md), [Backup and Restore](backup-and-restore.md), [Troubleshooting](troubleshooting.md) for detailed procedures.*