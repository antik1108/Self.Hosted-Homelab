# Troubleshooting Guide

## Quick Diagnostics

### Connectivity Matrix
| From \ To | Proxmox LAN (1.10) | Proxmox TS (100.122.224.81) | OpenWrt LAN (1.1) | OpenWrt via Jump |
|-----------|-------------------|----------------------------|-------------------|------------------|
| MacBook LAN | ✅ SSH, Web | ✅ SSH, Web | ✅ SSH, Web | ✅ SSH |
| MacBook TS | ✅ SSH, Web | ✅ SSH, Web | ❌ Direct | ✅ SSH |
| Proxmox | - | - | ✅ SSH | - |
| OpenWrt | ✅ Ping | ❌ Direct | - | - |

---

## Common Issues & Solutions

### 1. Tailscale Connection Issues

#### Symptoms
- `tailscale status` shows "Stopped" or "Disconnected"
- `tailscale ping 100.122.224.81` times out
- High latency via DERP relay

#### Diagnosis
```bash
# Check service
systemctl status tailscaled

# Check logs
journalctl -u tailscaled -f

# Check auth
tailscale status --json | jq '.BackendState'

# Network path
tailscale netcheck
```

#### Solutions
| Issue | Fix |
|-------|-----|
| Service stopped | `systemctl start tailscaled` |
| Auth expired | `tailscale up --authkey=NEW_KEY` |
| Firewall blocking | Check `ts-input` chain, UDP 41641 |
| DERP relay only | Check NAT type, enable UPnP, consider exit node |
| IPv6 issues | `tailscale set --accept-dns=false` (if DNS broken) |

---

### 2. SSH ProxyJump Failures

#### Symptoms
- `ssh openwrt` times out or "connection refused"
- "channel 0: open failed: connect failed"

#### Diagnosis
```bash
# Test Proxmox SSH first
ssh pve "echo OK"

# Test ProxyJump manually
ssh -J pve root@192.168.1.1 -v

# Check Proxmox AllowTcpForwarding
ssh pve "grep AllowTcpForwarding /etc/ssh/sshd_config"

# Check OpenWrt dropbear
ssh pve "ssh root@192.168.1.1 '/etc/init.d/dropbear status'"
```

#### Solutions
| Issue | Fix |
|-------|-----|
| Proxmox SSH down | `ssh pve "systemctl restart ssh"` |
| `AllowTcpForwarding no` | Set `AllowTcpForwarding yes` in `/etc/ssh/sshd_config` |
| Wrong key | Use `id_rsa` for OpenWrt, `id_ed25519` for Proxmox |
| Stale ControlMaster | `ssh -O exit pve && ssh -O exit openwrt` |
| OpenWrt dropbear stopped | `ssh pve "ssh root@192.168.1.1 '/etc/init.d/dropbear start'"` |

---

### 3. OpenWrt Unreachable

#### Symptoms
- Cannot SSH to 192.168.1.1
- LuCI not loading
- DHCP not working

#### Diagnosis
```bash
# From Proxmox
ssh pve "ping -c 3 192.168.1.1"
ssh pve "arp -n | grep 192.168.1.1"

# Check OpenWrt services
ssh pve "ssh root@192.168.1.1 '/etc/init.d/dropbear status && /etc/init.d/uhttpd status && /etc/init.d/dnsmasq status && /etc/init.d/firewall status'"

# Check logs
ssh pve "ssh root@192.168.1.1 'logread -e error | tail -20'"
```

#### Solutions
| Issue | Fix |
|-------|-----|
| Network down | `/etc/init.d/network restart` |
| Firewall blocking | Check `fw4 print`, ensure LAN zone `input: ACCEPT` |
| dropbear stopped | `/etc/init.d/dropbear start` |
| uhttpd stopped | `/etc/init.d/uhttpd start` |
| Overlay full | Clean ROM duplicates (see below) |
| Wrong IP | Check `uci show network.lan.ipaddr` |
| Bricked | TFTP recovery |

---

### 4. Proxmox Web UI Issues

#### Symptoms
- `https://192.168.1.10:8006` doesn't load
- `https://100.122.224.81:8006` doesn't load
- Certificate errors

#### Diagnosis
```bash
# Check service
ssh pve "systemctl status pveproxy"

# Check ports
ssh pve "ss -tulpn | grep 8006"

# Check firewall
ssh pve "pve-firewall simulate INPUT 192.168.1.50 tcp 8006"
ssh pve "pve-firewall simulate INPUT 100.122.224.81 tcp 8006"

# Check cert
ssh pve "openssl x509 -in /etc/pve/local/pve-ssl.pem -noout -dates"
```

#### Solutions
| Issue | Fix |
|-------|-----|
| pveproxy stopped | `systemctl restart pveproxy` |
| Firewall blocking | Check `host.fw` rules, `pve-firewall compile` |
| Cert expired | `pvecm updatecerts --force` |
| Wrong bind address | Check `/etc/default/pveproxy` for `LISTEN_IP` |

---

### 5. WWAN (Wi-Fi Upstream) Disconnected

#### Symptoms
- No internet from LAN
- `phy1-sta0` no IP or wrong IP
- `iwinfo phy1-sta0 info` shows "Not-Associated"

#### Diagnosis
```bash
# Check association
ssh openwrt "iwinfo phy1-sta0 info"

# Check wpa_supplicant
ssh openwrt "logread -e wpa_supplicant | tail -20"

# Check DHCP
ssh openwrt "ifstatus wwan"
ssh openwrt "cat /tmp/resolv.conf.d/resolv.conf.auto"
```

#### Solutions
| Issue | Fix |
|-------|-----|
| Wrong password | Update `vault_wifi_5g_password` in vault, re-run wireless playbook |
| Channel changed | Scan: `iwinfo phy1 scan`, update channel in config |
| Radio off | `wifi up radio1` |
| Country mismatch | Ensure `country='IN'` on radio1 |
| AP disabled (expected) | 5GHz AP must be disabled for STA on different channel |

---

### 6. Overlay Space Full (OpenWrt)

#### Symptoms
- `df -h /overlay` shows 93%+ used
- Cannot install packages
- Config changes not persisting
- `uci commit` errors

#### Diagnosis
```bash
ssh openwrt "df -h /overlay"
ssh openwrt "du -h /overlay/upper/* | sort -h"
```

#### Solutions
```bash
# Clean ROM duplicates (safe)
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

> **Never delete** `/overlay/upper/etc/` - contains your configs!

---

### 7. DNS Resolution Failures

#### Symptoms
- `nslookup google.com` fails
- `ping google.com` fails but `ping 1.1.1.1` works
- LAN hostnames not resolving

#### Diagnosis
```bash
# From Proxmox
ssh pve "nslookup google.com 192.168.1.1"
ssh pve "nslookup pve-host.lan 192.168.1.1"

# From OpenWrt
ssh openwrt "nslookup google.com 1.1.1.1"
ssh openwrt "cat /tmp/resolv.conf.d/resolv.conf.auto"

# Check dnsmasq
ssh openwrt "logread -e dnsmasq | tail -20"
```

#### Solutions
| Issue | Fix |
|-------|-----|
| Upstream DNS down | Change DNS servers in `/etc/config/dhcp` |
| Rebind protection | `rebind_protection=1` blocks private IPs; add `rebind_domain` if needed |
| `localservice=1` | dnsmasq only listens on LAN; check interface binding |
| No upstream in resolv.conf.auto | Check WWAN DHCP, restart dnsmasq |

---

### 8. DHCP Not Working

#### Symptoms
- Clients get no IP or 169.254.x.x
- Static reservations not applied

#### Diagnosis
```bash
# Check dnsmasq
ssh openwrt "/etc/init.d/dnsmasq status"
ssh openwrt "logread -e dnsmasq | tail -20"

# Check leases
ssh openwrt "cat /tmp/dhcp.leases"

# Check config
ssh openwrt "uci show dhcp.lan"
```

#### Solutions
| Issue | Fix |
|-------|-----|
| dnsmasq stopped | `/etc/init.d/dnsmasq start` |
| Wrong interface | Check `dhcp.lan.interface='lan'` |
| Firewall blocking | Ensure LAN zone allows DHCP (67/68) |
| Static MAC mismatch | Verify MAC in `uci show dhcp` matches device |

---

### 9. Ansible Failures

#### OpenWrt: "python3 not found"
```bash
# Cause: OpenWrt has no Python
# Fix: Use raw module or community.openwrt.uci
# Wrong:
- name: Ping
  ansible.builtin.ping:

# Correct:
- name: Check system
  ansible.builtin.raw: "ubus call system board"
```

#### OpenWrt: "Could not match supplied host pattern"
```bash
# Cause: inventory group name mismatch
# Fix: Check inventory/hosts.yml group names match playbook hosts:
# playbooks/openwrt.yml: hosts: main-router
# inventory/hosts.yml: openwrt: hosts: main-router:
```

#### Proxmox: "Failed to connect"
```bash
# Cause: Tailscale down or wrong IP
# Fix:
tailscale status
# Update ansible_host in inventory/hosts.yml
```

#### Vault: "Decryption failed"
```bash
# Cause: Wrong password or corrupted vault
# Fix:
ansible-vault view secrets/vault.yml --vault-password-file secrets/vault_pass.txt
# If fails: check vault_pass.txt content, recreate vault
```

---

### 10. Certificate Warnings

#### Symptoms
- Browser shows "Not Secure" for LuCI and Proxmox
- `curl -k` required

#### Explanation
Self-signed certificates are used for internal services. This is **expected and acceptable** for a homelab.

#### Mitigation
```bash
# Verify fingerprint on first connect
ssh pve "openssl x509 -in /etc/pve/local/pve-ssl.pem -noout -fingerprint -sha256"
ssh openwrt "openssl x509 -in /etc/uhttpd.crt -noout -fingerprint -sha256"

# Add to browser exception list
# Or use mkcert for local CA (advanced)
```

---

## Diagnostic Commands Reference

### Proxmox
```bash
# System
systemctl status pveproxy pvedaemon ssh tailscaled networking
journalctl -u pveproxy -u ssh -u tailscaled --since '1 hour ago'

# Network
ip -br a; ip route; ss -tulpn
pve-firewall status; pve-firewall compile

# Storage
df -h; lvs; pvesm status

# Tailscale
tailscale status; tailscale ping 100.110.79.92; tailscale netcheck
```

### OpenWrt
```bash
# System
logread -e error; dmesg -T | tail; uptime; free; df -h

# Network
ip -br a; ip route; netstat -tulpn
fw4 print; nft list ruleset

# Services
/etc/init.d/dropbear status; /etc/init.d/uhttpd status
/etc/init.d/dnsmasq status; /etc/init.d/firewall status
/etc/init.d/network status

# Wireless
iwinfo; iwinfo wlan0 assoclist; iwinfo phy1-sta0 info
wifi status

# DHCP/DNS
cat /tmp/dhcp.leases; cat /tmp/resolv.conf.d/resolv.conf.auto
```

### Tailscale (Control Machine)
```bash
tailscale status; tailscale ping <IP>
tailscale netcheck; tailscale file cp
```

---

## Escalation Path

1. **Run diagnostics** above
2. **Check logs** (`journalctl`, `logread`)
3. **Test isolation** (firewall stop, direct LAN access)
4. **Restore from backup** (see backup-and-restore.md)
5. **Re-provision via Ansible** (full config re-apply)
6. **Hardware recovery** (TFTP for OpenWrt, reinstall for Proxmox)

---

*See [Daily Operations](daily-operations.md) for routine checks, [Network Management](network-management.md) for network issues, [Firewall Management](firewall-management.md) for firewall issues, [Backup and Restore](backup-and-restore.md) for recovery.*