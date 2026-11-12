# Stage D: Proxmox Security Investigation Report
**Date:** 2026-11-12  
**Status:** Read-Only Investigation Complete

---

## Executive Summary

Investigated three security areas on Proxmox VE (homelab, 100.122.224.81 / 192.168.1.10):
1. **pveproxy** - Proxmox Web UI/API listening on all interfaces (*:8006)
2. **rpcbind** - RPC portmapper exposed on all interfaces (0.0.0.0:111, [::]:111)
3. **Proxmox Firewall** - Disabled, no rules configured

---

## 1. pveproxy Investigation

### Current Configuration
| Parameter | Value | Source |
|-----------|-------|--------|
| Listen Address | `*:8006` (all IPv4 + IPv6) | Default (no `/etc/default/pveproxy`) |
| TLS | Enabled (self-signed cert) | Default |
| Workers | 3 | Default |
| Accessible via | LAN (192.168.1.10), Tailscale (100.122.224.81) | Verified |

### Verification Evidence
```bash
# From MacBook via Tailscale
$ curl -k -s -o /dev/null -w '%{http_code}' https://100.122.224.81:8006
200

# From Proxmox via LAN
$ curl -k -s -o /dev/null -w '%{http_code}' https://192.168.1.10:8006
200
```

### Risk Assessment
| Risk | Likelihood | Impact | Notes |
|------|------------|--------|-------|
| LAN exposure | High | Medium | Any device on LAN can access Proxmox UI |
| Tailscale exposure | Low | Low | Only tailnet devices (4 registered) |
| Credential theft | Medium | High | Self-signed cert, no MFA configured |

### Configuration Options
File: `/etc/default/pveproxy` (doesn't exist, uses defaults)
```bash
# To bind to specific interfaces:
LISTEN_IP="192.168.1.10 100.122.224.81"
# OR bind to LAN only (break Tailscale access):
LISTEN_IP="192.168.1.10"
```

### Impact of Changes
| Change | LAN Access | Tailscale Access | SSH/ProxyJump |
|--------|------------|------------------|---------------|
| `LISTEN_IP="192.168.1.10 100.122.224.81"` | ✅ | ✅ | ✅ |
| `LISTEN_IP="192.168.1.10"` | ✅ | ❌ | ✅ |
| Keep default (`*:8006`) | ✅ | ✅ | ✅ |

---

## 2. rpcbind Investigation

### Current Configuration
| Parameter | Value |
|-----------|-------|
| Listen Address | `0.0.0.0:111` (IPv4), `[::]:111` (IPv6) |
| Services Registered | Only portmapper (100000 v2/v3/v4) |
| systemd Status | Enabled, running since Apr 2026 |
| Dependencies | None (only multi-user.target) |

### Verification Evidence
```bash
$ rpcinfo -p localhost
program vers proto   port  service
100000    4   tcp    111  portmapper
100000    3   tcp    111  portmapper
100000    2   tcp    111  portmapper
100000    4   udp    111  portmapper
100000    3   udp    111  portmapper
100000    2   udp    111  portmapper

$ systemctl list-dependencies rpcbind.service --reverse
rpcbind.service
● └─multi-user.target
  └─graphical.target
```

### Risk Assessment
| Risk | Likelihood | Impact | Notes |
|------|------------|--------|-------|
| Information disclosure | Medium | Low | Only portmapper visible, no NFS/mountd |
| Amplification attacks | Low | Low | UDP portmapper can amplify |
| Lateral movement | Low | Medium | Recon for NFS exports |

### Dependencies Check
- **No NFS services** running (nfs-server, nfs-kernel-server not installed)
- **No Proxmox storage** uses NFS (only local-lvm, local, HDD-lvm)
- **No VMs/CTs** running that might need NFS
- **Safe to disable** - no registered RPC services beyond portmapper

---

## 3. Proxmox Firewall Investigation

### Current Status
| Component | Status |
|-----------|--------|
| pve-firewall service | Disabled/running (status: disabled/running) |
| Cluster firewall (`/etc/pve/firewall/cluster.fw`) | Not configured |
| Host firewall (`/etc/pve/nodes/homelab/host.fw`) | Not configured |
| Firewall directory (`/etc/pve/firewall/`) | Empty |

### Required Access to Preserve
| Service | Port | Interface | Purpose |
|---------|------|-----------|---------|
| SSH | 22 | Tailscale + LAN | Admin access, ProxyJump |
| Tailscale | 41641/UDP | WAN | Overlay network |
| Tailscale API | 42522/36222 | tailscale0 | Control plane |
| pveproxy | 8006 | LAN + Tailscale | Web UI/API |
| spiceproxy | 3128 | LAN + Tailscale | VM console |
| DNS | 53 | LAN | Upstream via OpenWrt |
| NTP | 123 | LAN + WAN | Time sync |
| ICMP | - | All | Connectivity checks |

---

## Consolidated Findings & Recommendations

### Changes Actually Necessary (Security Benefit > Operational Risk)

| # | Change | Security Benefit | Operational Risk | Recommendation |
|---|--------|------------------|------------------|----------------|
| 1 | **Configure pveproxy LISTEN_IP** | Restricts management UI to authorized networks only | Low (if both IPs included) | ✅ **DO** - Bind to LAN + Tailscale |
| 2 | **Disable rpcbind** | Removes unnecessary exposed service | None (no NFS in use) | ✅ **DO** |
| 3 | **Configure spiceproxy LISTEN_IP** | Consistent with pveproxy | Low | ✅ **DO** - Same IPs |
| 4 | **Enable Proxmox firewall** | Host-level packet filtering | Medium (must get rules right) | ⚠️ **CONDITIONAL** - Only with reviewed ruleset |

### Changes NOT Necessary (No Real Benefit)

| Change | Reason |
|--------|--------|
| Bind pveproxy to LAN only | Breaks Tailscale remote management - no compensating benefit |
| Change SSH to key-only | Already using ed25519 key; password auth not exposed externally |
| Restrict SSH to Tailscale only | Breaks ProxyJump for OpenWrt management |

---

## Proposed Implementation Plan

### Phase D1: pveproxy & spiceproxy Binding (Low Risk)
1. Create `/etc/default/pveproxy` with `LISTEN_IP="192.168.1.10 100.122.224.81"`
2. Create `/etc/default/spiceproxy` with same `LISTEN_IP`
3. Restart both services
4. Verify access from LAN and Tailscale

### Phase D2: Disable rpcbind (Zero Risk)
1. `systemctl stop rpcbind rpcbind.socket`
2. `systemctl disable rpcbind rpcbind.socket`
2. Verify no service disruption

### Phase D3: Proxmox Firewall (Conditional - Requires Approval)
1. Create minimal host.fw ruleset
2. Enable pve-firewall
3. Test all access paths
4. Only proceed if Phases D1-D2 complete successfully

---

## Risk Matrix for Proposed Changes

| Change | Rollback Complexity | Blast Radius | Test Plan |
|--------|---------------------|--------------|-----------|
| pveproxy LISTEN_IP | Low (delete file, restart) | Management UI only | curl from LAN + Tailscale |
| spiceproxy LISTEN_IP | Low | VM console only | noVNC from LAN + Tailscale |
| rpcbind disable | Low (systemctl enable/start) | None | Verify no NFS mounts |
| pve-firewall enable | Medium (disable service) | All host traffic | Full connectivity matrix |

---

## Approval Requested

**Please approve the following:**

1. **pveproxy binding** - Create `/etc/default/pveproxy` with `LISTEN_IP="192.168.1.10 100.122.224.81"`
2. **spiceproxy binding** - Create `/etc/default/spiceproxy` with same `LISTEN_IP`
3. **rpcbind disable** - Stop and disable rpcbind service and socket
4. **Proxmox firewall** - Defer until after D1-D2 verified; present ruleset for separate approval

**Do not proceed with any changes until approved.**

---
*End of Stage D Investigation Report*