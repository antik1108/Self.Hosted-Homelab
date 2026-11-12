# Stage D: Proxmox Security - Execution Summary & Firewall Proposal
**Date:** 2026-11-12  
**Status:** rpcbind Removal Complete | Firewall Ruleset Prepared (Pending Approval)

---

## 1. rpcbind Removal — ✅ COMPLETE

### Verification Evidence

| Check | Result |
|-------|--------|
| rpcbind.service status | `masked`, `inactive (dead)` |
| rpcbind.socket status | `masked`, `inactive (dead)` |
| Port 111 (IPv4) | **Not listening** |
| Port 111 (IPv6) | **Not listening** |
| rpcinfo -p localhost | `Connection refused` |
| nfs-blkmap.service | Still running (independent, no NFS mounts) |
| Proxmox storage | No NFS - only local-lvm, local, hdd-storage |

### Connectivity After Removal ✅

```bash
# Tailscale
$ tailscale status
100.122.224.81  homelab  linux    active
100.110.79.92   antiks-macbook-pro  macOS    active; relay "bom"

# Proxmox Web UI via Tailscale
$ curl -k https://100.122.224.81:8006  → 200 OK

# Proxmox Web UI via LAN
$ curl -k https://192.168.1.10:8006  → 200 OK

# SSH via Tailscale (ed25519)
$ ssh root@100.122.224.81  → Working

# MacBook → Proxmox → OpenWrt ProxyJump
$ ssh -J root@100.122.224.81 root@192.168.1.1  → Working
```

### Backup Created
- `/root/backup-rpcbind-2026-11-12/` (service configs)

### Rollback Procedure
```bash
systemctl unmask rpcbind rpcbind.socket
systemctl enable rpcbind rpcbind.socket
systemctl start rpcbind rpcbind.socket
```

---

## 2. pveproxy/spiceproxy Binding — HOLD (As Requested)

### Finding: LISTEN_IP Supports Single IP Only

The Proxmox `create_reusable_socket()` function accepts a single `$host` parameter. Multiple space-separated IPs in `/etc/default/pveproxy` are **not supported**.

### Alternative: Proxmox Firewall Rules (Prepared Below)

The firewall ruleset below achieves the same security goal:
- pveproxy/spiceproxy continue listening on `*:8006` and `*:3128`
- Firewall restricts access to only LAN (192.168.1.0/24) and Tailscale (100.122.224.81/32)

---

## 3. Proxmox Firewall Ruleset — READY FOR APPROVAL

### Configuration Files

**`/etc/pve/firewall/cluster.fw`** (Cluster-wide):
```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
policy_forward: ACCEPT

[ALIASES]
local_network 192.168.1.0/24
```

**`/etc/pve/nodes/homelab/host.fw`** (Host-specific):
```ini
[RULES]
# SSH from LAN and Tailscale
IN SSH(ACCEPT) -source 192.168.1.0/24
IN SSH(ACCEPT) -source 100.122.224.81/32

# Tailscale UDP (DERP/relay) from anywhere
IN ACCEPT -p udp -dport 41641

# Tailscale control plane TCP from Tailscale IP
IN ACCEPT -p tcp -dport 42522 -source 100.122.224.81/32
IN ACCEPT -p tcp -dport 36222 -source 100.122.224.81/32

# Proxmox Web UI (8006) from LAN and Tailscale
IN ACCEPT -p tcp -dport 8006 -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 8006 -source 100.122.224.81/32

# SPICE proxy (3128) from LAN and Tailscale
IN ACCEPT -p tcp -dport 3128 -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 3128 -source 100.122.224.81/32

# DNS from LAN (for VMs/containers)
IN ACCEPT -p udp -dport 53 -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 53 -source 192.168.1.0/24

# NTP from anywhere
IN ACCEPT -p udp -dport 123

# Default policies
OUT ACCEPT
FORWARD ACCEPT
```

### Simulation Test Results ✅

| Source | Destination | Port | Protocol | Action |
|--------|-------------|------|----------|--------|
| 100.122.224.81 (Tailscale) | host | 8006 | tcp | **ACCEPT** |
| 100.122.224.81 (Tailscale) | host | 22 | tcp | **ACCEPT** |
| 100.122.224.81 (Tailscale) | host | 3128 | tcp | **ACCEPT** |
| 100.122.224.81 (Tailscale) | host | 42522 | tcp | **ACCEPT** |
| 100.122.224.81 (Tailscale) | host | 36222 | tcp | **ACCEPT** |
| 1.2.3.4 (any) | host | 41641 | udp | **ACCEPT** |
| 192.168.1.50 (LAN) | host | 8006 | tcp | **ACCEPT** |
| 192.168.1.50 (LAN) | host | 22 | tcp | **ACCEPT** |
| 10.0.0.1 (external) | host | 8006 | tcp | **DROP** |
| 10.0.0.1 (external) | host | 22 | tcp | **DROP** |
| 10.0.0.1 (external) | host | 3128 | tcp | **DROP** |

### Preserved Access
| Service | LAN (192.168.1.0/24) | Tailscale (100.122.224.81) |
|---------|---------------------|---------------------------|
| SSH (22) | ✅ | ✅ |
| Proxmox Web UI (8006) | ✅ | ✅ |
| SPICE Proxy (3128) | ✅ | ✅ |
| Tailscale UDP (41641) | ✅ (any) | ✅ |
| Tailscale Control (42522, 36222) | ❌ | ✅ |
| DNS (53) | ✅ | ❌ |
| NTP (123) | ✅ | ✅ |
| ICMP/ping | ✅ (default) | ✅ (default) |
| VM/Container Forwarding | ✅ | ✅ |

### Cluster Communication
- `local_network` alias explicitly set to `192.168.1.0/24`
- Default rules allow corosync (5405-5412), cluster API, SSH between nodes
- Single-node cluster: no inter-node traffic needed

---

## 4. Approval Requested

### Approve to Enable Firewall?
- [ ] **YES** - Enable Proxmox firewall with the above ruleset
- [ ] **NO** - Keep firewall disabled, rules remain staged

### If YES, Execution Plan:
1. `pve-firewall start` (applies iptables rules)
2. Verify all connectivity (Tailscale, SSH, Web UI, ProxyJump)
3. Run simulation tests again
4. Document any issues

### Rollback Plan (if issues):
```bash
pve-firewall stop  # Removes all Proxmox iptables rules immediately
```

---

## 5. Next Steps After Firewall Approval

1. **Complete SSH/Ansible Verification** - Test Ansible playbooks in check mode
2. **Backup/Disaster Recovery Documentation** - Finalize recovery procedures
3. **End-to-End Network Validation** - Full connectivity matrix
4. **Documentation Update** - All docs + CHANGELOG
5. **Git Commit** - Review and commit all changes

---
*End of Stage D Summary*