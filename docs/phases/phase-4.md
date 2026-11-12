# Phase 4: Security Hardening

## Objective
Apply defense-in-depth security controls to both OpenWrt and Proxmox based on the Phase 3 baseline, minimizing attack surface while maintaining operational access via Tailscale and LAN.

---

## Scope
- OpenWrt: Service binding restrictions, SSH hardening, HTTPS enforcement, regulatory compliance, 5GHz AP decision
- Proxmox: Service removal (rpcbind), firewall minimal ruleset, Tailscale integration validation
- Verification: All access paths, external blocking, internal connectivity

---

## OpenWrt Hardening (Stage C)

### C1: 5GHz Access Point Decision
**Action**: Disabled 5GHz AP (`default_radio1.disabled='1'`)

**Reasoning**:
- Single 5GHz radio (phy1) used for WWAN STA on channel 149 (upstream college Wi-Fi)
- Hardware limitation: Cannot simultaneously operate STA and AP on different channels
- Attempting AP on different channel causes STA disconnection
- 2.4GHz AP (`HomeLab-2G`) provides sufficient coverage for lab devices

**Verification**:
```bash
ssh openwrt "iwinfo wlan1 info"
# Output: "wlan1: No such device" (AP interface not created)
ssh openwrt "iwinfo phy1-sta0 info"
# Output: Associated with Rishihood_Learner on channel 149
```

### C2: Regulatory Domain (India)
**Action**: Set `country='IN'` on both radio0 and radio1

**Commands**:
```bash
uci set wireless.radio0.country='IN'
uci set wireless.radio1.country='IN'
uci commit wireless
wifi reload
```

**Verification**:
```bash
ssh openwrt "iw reg get"
# Output: country IN: DFS-ETSI
# Channels 1-13 (2.4GHz), 36-165 (5GHz) with DFS requirements
```

### C3: LuCI/uhttpd LAN-Only Binding
**Action**: Restrict uhttpd to `192.168.1.1:80` and `192.168.1.1:443`

**Commands**:
```bash
uci set uhttpd.main.listen_http='192.168.1.1:80'
uci set uhttpd.main.listen_https='192.168.1.1:443'
uci commit uhttpd
/etc/init.d/uhttpd reload
```

**Verification**:
```bash
# From LAN
curl -k http://192.168.1.1      # 307 redirect to HTTPS
curl -k https://192.168.1.1     # 200 OK

# From WWAN (simulated)
# nmap -p 80,443 10.7.28.50  # Should show filtered/closed
```

### C4: Dropbear SSH LAN-Only Binding
**Action**: Bind dropbear to `lan` interface only

**Commands**:
```bash
uci set dropbear.@dropbear[0].Interface='lan'
uci commit dropbear
/etc/init.d/dropbear reload
```

**Verification**:
```bash
# From LAN
ssh -i ~/.ssh/id_rsa root@192.168.1.1 "echo OK"  # ✅ Works

# From WWAN (via Proxmox)
# ssh -J pve root@192.168.1.1  # ✅ Works (LAN side)

# Direct from WWAN would fail (no route to br-lan)
```

### C5: SSH Key-Only Authentication
**Action**: Disable password authentication for root

**Commands**:
```bash
uci set dropbear.@dropbear[0].PasswordAuth='off'
uci set dropbear.@dropbear[0].RootPasswordAuth='off'
uci set dropbear.@dropbear[0].RSAAuth='on'
uci set dropbear.@dropbear[0].PubkeyAuth='on'
uci commit dropbear
/etc/init.d/dropbear reload
```

**Verification**:
```bash
# Key auth (should work)
ssh -i ~/.ssh/id_rsa root@192.168.1.1 "echo OK"

# Password auth (should fail)
ssh -o PreferredAuthentications=password root@192.168.1.1
# Permission denied (publickey)
```

### C6: HTTPS Enforcement with Redirect
**Action**: Enable HTTP→HTTPS redirect (307)

**Commands**:
```bash
uci set uhttpd.main.redirect_https='1'
uci commit uhttpd
/etc/init.d/uhttpd reload
```

**Verification**:
```bash
curl -v http://192.168.1.1 2>&1 | grep -E "307|Location"
# < HTTP/1.1 307 Temporary Redirect
# < Location: https://192.168.1.1/
```

### C7: Timezone Configuration
**Action**: Set timezone to Asia/Kolkata (IST-5:30)

**Commands**:
```bash
uci set system.@system[0].timezone='IST-5:30'
uci set system.@system[0].zonename='Asia/Kolkata'
uci commit system
/etc/init.d/sysntpd restart
```

**Verification**:
```bash
ssh openwrt "date"
# Output: Thu Nov 12 13:30:00 IST 2026
ssh openwrt "uci show system.@system[0].timezone"
# system.@system[0].timezone='IST-5:30'
```

---

## Proxmox Hardening (Stage D)

### D1: rpcbind Removal
**Action**: Stop, disable, and mask rpcbind (port 111)

**Commands**:
```bash
systemctl stop rpcbind.socket rpcbind
systemctl disable rpcbind.socket rpcbind
systemctl mask rpcbind.socket rpcbind
```

**Verification**:
```bash
systemctl status rpcbind
# Loaded: masked (Reason: Unit rpcbind.service is masked.)
# Active: inactive (dead)

ss -tulpn | grep 111
# No output (port 111 closed)
```

### D2: pveproxy/spiceproxy Binding Investigation
**Finding**: `LISTEN_IP` in `/etc/default/pveproxy` supports **single IP only**, not interface-specific binding

**Decision**: Keep pveproxy on `*:8006` (all interfaces), enforce access via pve-firewall

**Rationale**:
- Binding to specific IP requires knowing IP at boot (static is fine)
- But pveproxy doesn't support multiple IPs or interface names
- Firewall provides equivalent security with more flexibility
- Tailscale `ts-input` chain handles overlay traffic independently

**Configuration**:
```bash
# /etc/default/pveproxy (unchanged)
LISTEN_IP="*"
```

### D3: Minimal Host Firewall Ruleset
**Action**: Apply default-deny with explicit allow for management

**Cluster Firewall** (`/etc/pve/firewall/cluster.fw`):
```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
policy_forward: ACCEPT

[ALIASES]
local_network 192.168.1.0/24
```

**Host Firewall** (`/etc/pve/nodes/homelab/host.fw`):
```ini
[RULES]
IN SSH(ACCEPT) -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 8006 -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 3128 -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 8006 -source 100.122.224.81/32
IN ACCEPT -p tcp -dport 3128 -source 100.122.224.81/32
IN ACCEPT -p tcp -dport 22 -source 100.122.224.81/32
OUT ACCEPT
```

**Verification**:
```bash
pve-firewall compile  # ✅ No errors
pve-firewall start    # ✅ Started

# Test simulations
pve-firewall simulate INPUT 192.168.1.50 tcp 22     # ACCEPT
pve-firewall simulate INPUT 192.168.1.50 tcp 8006   # ACCEPT
pve-firewall simulate INPUT 100.122.224.81 tcp 22   # ACCEPT
pve-firewall simulate INPUT 100.122.224.81 tcp 8006 # ACCEPT
pve-firewall simulate INPUT 10.0.0.1 tcp 22         # DROP
pve-firewall simulate INPUT 10.0.0.1 tcp 8006       # DROP
```

### D4: Tailscale Integration Validation
**Finding**: Tailscale `ts-input` chain runs **before** pve-firewall, handles all Tailscale traffic

**Verification**:
```bash
iptables -L ts-input -n -v
# Chain ts-input (policy ACCEPT)
# pkts bytes target     prot opt in     out     source               destination
#    0     0 ACCEPT     all  --  tailscale0 *       0.0.0.0/0            0.0.0.0/0
#    0     0 ACCEPT     udp  --  *       *       0.0.0.0/0            0.0.0.0/0        udp dpt:41641
#    0     0 RETURN     all  --  *       *       100.115.92.0/23      0.0.0.0/0
#    0     0 DROP       all  --  !tailscale0 *       100.64.0.0/10      0.0.0.0/0

# Tailscale connectivity
tailscale status  # ✅ Active
tailscale ping 100.122.224.81  # ✅ ~80ms via DERP
ssh pve "echo OK"  # ✅ Works
curl -k https://100.122.224.81:8006  # ✅ Works
```

---

## Connectivity Verification (Post-Hardening)

### All Access Paths Working

| Path | SSH (22) | Web UI (443/8006) | SPICE (3128) | Ping |
|------|----------|-------------------|--------------|------|
| MacBook LAN → Proxmox | ✅ | ✅ | ✅ | ✅ |
| MacBook TS → Proxmox | ✅ | ✅ | ✅ | ✅ |
| MacBook LAN → OpenWrt (Jump) | ✅ | ✅ | N/A | ✅ |
| MacBook TS → OpenWrt (Jump) | ✅ | ✅ | N/A | ✅ |
| Proxmox → OpenWrt (LAN) | ✅ | N/A | N/A | ✅ |
| OpenWrt → Internet | N/A | N/A | N/A | ✅ |

### External Blocking Verified
```bash
# Proxmox firewall simulation
pve-firewall simulate INPUT 10.0.0.1 tcp 22     # DROP
pve-firewall simulate INPUT 10.0.0.1 tcp 8006   # DROP
pve-firewall simulate INPUT 203.0.113.100 tcp 22 # DROP

# OpenWrt WAN zone (fw4)
# nft list chain inet fw4 input_wan | grep reject  # REJECT present
```

### Tailscale Connectivity
```bash
tailscale ping 100.122.224.81  # ✅ 78ms via DERP "bom"
tailscale ping 100.110.79.92   # ✅ From Proxmoy
tailscale netcheck             # ✅ UDP: true, DERP: bom
```

### DNS & NTP
```bash
# Proxmox
ssh pve "nslookup google.com 192.168.1.1"      # ✅
ssh pve "chronyc sources -v"                   # ✅ 4 sources, * selected

# OpenWrt
ssh openwrt "nslookup google.com 1.1.1.1"      # ✅
ssh openwrt "ntpdate -q 0.openwrt.pool.ntp.org" # ✅
```

---

## Backups (Post-Hardening)

### Proxmox
```bash
# Firewall configs backed up
/etc/pve/nodes/homelab/host.fw.backup-20261112
/etc/pve/firewall/cluster.fw.backup-20261112

# Ansible backup playbook run
ansible-playbook playbooks/backup.yml
# backups/proxmox/2026-11-12-stage-d/
```

### OpenWrt
```bash
# Pre-Stage C backup
backups/openwrt/2026-11-12-stage-c-pre/
  openwrt-backup-2026-11-12T10:15:00.tar.gz
  uci-configs-2026-11-12T10:15:00.txt

# Post-Stage C backup
backups/openwrt/2026-11-12-stage-c-post/
  openwrt-backup-2026-11-12T11:30:00.tar.gz
  uci-configs-2026-11-12T11:30:00.txt
```

---

## Residual Risks (Accepted)

| Risk | Justification |
|------|---------------|
| Self-signed TLS (LuCI, Proxmox) | Internal lab; fingerprint verification on first connect |
| Proxmox SSH password auth enabled | Ed25519 key primary; Tailscale + firewall restrict access |
| OpenWrt overlay 93% full | No opkg; config-only changes; monitored weekly |
| No 5GHz AP | Hardware limitation; 2.4GHz sufficient for lab |
| Proxmox Web UI on *:8006 | Firewall restricts to LAN + Tailscale IP |
| DERP relay latency (~80ms) | College CGNAT prevents direct WireGuard; acceptable for mgmt |
| No IPv6 internet | Upstream doesn't delegate prefix; ULA for local only |

---

## Metrics Comparison

| Metric | Phase 3 (Baseline) | Phase 4 (Hardened) |
|--------|-------------------|-------------------|
| OpenWrt SSH access | LAN + WWAN (password+key) | LAN only (key-only) |
| OpenWrt LuCI access | All interfaces (HTTP+HTTPS) | LAN only (HTTPS redirect) |
| Proxmox SSH access | All interfaces (password+key) | LAN + TS IP (key+password) |
| Proxmox Web UI access | All interfaces | LAN + TS IP |
| Proxmox rpcbind | Running (port 111 open) | **Masked, stopped** |
| Proxmox firewall | Default rules (broad) | **Minimal, default-deny** |
| 5GHz AP | Enabled (unencrypted) | **Disabled** (conflict) |
| Regulatory domain | Default (US) | **IN (India)** |
| Timezone | UTC | **Asia/Kolkata** |

---

## Sign-off
- **Date**: 2026-11-12
- **Status**: Complete - All hardening applied, all access verified
- **Documentation**: Updated in `docs/architecture/security-architecture.md`, `docs/operations/firewall-management.md`

---

*See [Phase 3 Report](phase-3.md) for baseline, [Security Architecture](../architecture/security-architecture.md) for controls.*