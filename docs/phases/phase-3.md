# Phase 3: Infrastructure Baseline

## Objective
Establish a working, documented baseline of the homelab infrastructure before security hardening (Phase 4).

---

## Scope
- Proxmox host installation and initial configuration
- OpenWrt router installation and initial configuration
- Network connectivity (LAN, WWAN, Tailscale)
- Ansible automation framework
- Backup procedures

---

## Completed Tasks

### Proxmox Host
- [x] Proxmox VE 9.2.2 installed on Dell i3-M380
- [x] Static IP `192.168.1.10/24` on `vmbr0`
- [x] Enterprise repo disabled, no-subscription repo enabled
- [x] System updated (`apt full-upgrade`)
- [x] Tailscale installed and authenticated (`100.122.224.81`)
- [x] SSH key authentication (Ed25519) configured
- [x] Basic pve-firewall enabled with default rules
- [x] Storage configured: local (dir), local-lvm (lvmthin), hdd-storage (dir)

### OpenWrt Router
- [x] OpenWrt 25.12.6 (snapshot) flashed on TP-Link Archer C20 v5
- [x] LAN interface: `192.168.1.1/24` on `br-lan` (VLAN 1)
- [x] WAN interface: DHCP on `eth0.2` (VLAN 2) - unused
- [x] WWAN interface: DHCP on `phy1-sta0` (5GHz radio1, channel 149)
- [x] 2.4GHz AP: `HomeLab-2G` on radio0, channel 1, WPA2-CCMP
- [x] 5GHz AP: Disabled (channel conflict with STA)
- [x] DHCP server: 192.168.1.100-250, 12h lease
- [x] Static reservations: Proxmox (`.10`), MacBook (`.50`)
- [x] DNS: dnsmasq with upstream 1.1.1.1, 8.8.8.8, 9.9.9.9
- [x] Firewall: Default zones (LAN accept, WAN reject)
- [x] SSH (dropbear) on port 22, password auth enabled
- [x] LuCI (uhttpd) on port 80/443, HTTP→HTTPS redirect
- [x] Timezone: Asia/Kolkata (IST-5:30)
- [x] Regulatory domain: IN (India) on both radios

### Network Connectivity
- [x] LAN: Proxmox ↔ OpenWrt ↔ MacBook (when on LAN)
- [x] WWAN: OpenWrt → College Wi-Fi (`Rishihood_Learner`) → Internet
- [x] Tailscale: MacBook ↔ Proxmox (DERP relay via Mumbai)
- [x] ProxyJump: MacBook → Proxmox (Tailscale) → OpenWrt (LAN)
- [x] DNS resolution: LAN hostnames, internet domains
- [x] NTP sync: Both hosts

### Ansible Automation
- [x] Inventory with ProxyJump configuration
- [x] Vault with encrypted secrets (WiFi passwords, SSH pubkey, Proxmox password)
- [x] `community.openwrt` collection for UCI management
- [x] Playbooks: verify.yml, backup.yml, openwrt.yml, proxmox.yml
- [x] Roles: openwrt (network, wireless, dhcp, firewall, ssh, luci, system), proxmox (network, firewall, storage, packages, tailscale)

### Backup Procedures
- [x] Proxmox: network interfaces, firewall configs
- [x] OpenWrt: sysupgrade backup, UCI config dump
- [x] Automated via `playbooks/backup.yml`
- [x] Stored in `backups/proxmox/YYYY-MM-DD/`, `backups/openwrt/YYYY-MM-DD/`

---

## Key Metrics (Baseline)

| Metric | Value |
|--------|-------|
| **Proxmox RAM** | 3.6 GB total, ~1.2 GB used |
| **Proxmox SSD** | 238 GB, ~15 GB used |
| **Proxmox HDD** | 465 GB, ~5 GB used |
| **OpenWrt Overlay** | 960 KB total, ~72 KB free (93% used) |
| **OpenWrt RAM** | 128 MB total, ~45 MB free |
| **Tailscale Latency** | ~80-120 ms (DERP relay) |
| **WWAN Speed** | ~50-100 Mbps (college Wi-Fi) |
| **LAN Speed** | 1 Gbps (physical) |

---

## Known Issues (Carried to Phase 4)

| Issue | Severity | Phase 4 Action |
|-------|----------|----------------|
| OpenWrt overlay 93% full | High | Clean ROM duplicates, monitor |
| No opkg/Python3 on OpenWrt | Medium | Use raw/uci modules only |
| 5GHz AP disabled (channel conflict) | Medium | Accept until router replacement |
| Proxmox Web UI on *:8006 | Low | Restrict via firewall |
| Self-signed TLS certs | Low | Accept for lab |
| rpcbind running (port 111) | Medium | Remove in Phase 4 |
| Dropbear password auth enabled | Medium | Disable in Phase 4 |
| uhttpd listening on all interfaces | Medium | Bind to LAN only in Phase 4 |
| No IPv6 internet (ULA only) | Low | Accept |
| DERP relay latency | Low | Accept |

---

## Verification Results

### Ansible Verification
```bash
ansible-playbook playbooks/verify.yml --vault-password-file secrets/vault_pass.txt
# ✅ Proxmox: ping OK, facts gathered
# ✅ OpenWrt: raw module works, system board returned
```

### Connectivity Tests
| Test | Result |
|------|--------|
| MacBook → Proxmox (Tailscale SSH) | ✅ |
| MacBook → Proxmox (Tailscale Web UI) | ✅ |
| MacBook → Proxmox (LAN SSH) | ✅ |
| MacBook → Proxmox (LAN Web UI) | ✅ |
| MacBook → OpenWrt (ProxyJump SSH) | ✅ |
| MacBook → OpenWrt (ProxyJump Web UI) | ✅ |
| Proxmox → OpenWrt (LAN SSH) | ✅ |
| OpenWrt → Internet (WWAN) | ✅ |
| DNS resolution (LAN + Internet) | ✅ |
| NTP sync (both hosts) | ✅ |

### Backup Verification
```bash
# Proxmox backups exist
ls backups/proxmox/2026-11-12/
# interfaces-*.bak, host.fw-*.bak, cluster.fw-*.bak

# OpenWrt backups exist
ls backups/openwrt/2026-11-12/
# openwrt-backup-*.tar.gz, uci-configs-*.txt

# Sysupgrade backup extracts
tar -tzf backups/openwrt/2026-11-12/openwrt-backup-*.tar.gz | head
# etc/config/, etc/dropbear/, etc/uhttpd.*, etc/ssl/...
```

---

## Documentation Created (Phase 3)
- `docs/architecture/overview.md`
- `docs/architecture/network-topology.md`
- `docs/architecture/hardware-inventory.md`
- `docs/architecture/ip-addressing.md`
- `docs/architecture/security-architecture.md`
- `docs/setup/prerequisites.md`
- `docs/setup/proxmox-setup.md`
- `docs/setup/openwrt-setup.md`
- `docs/setup/remote-access-setup.md`
- `docs/setup/ansible-setup.md`
- `docs/operations/daily-operations.md`
- `docs/operations/network-management.md`
- `docs/operations/firewall-management.md`
- `docs/operations/backup-and-restore.md`
- `docs/operations/troubleshooting.md`

---

## Sign-off
- **Date**: 2026-11-12
- **Status**: Complete - Ready for Phase 4
- **Next Phase**: Security Hardening (Phase 4)

---

*See [Phase 4 Report](phase-4.md) for security hardening details.*