# Architecture Decision Records (ADRs)

## ADR-001: Tailscale as Primary Remote Access
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Need secure remote management from college dorm / anywhere without port forwarding.
**Decision**: Use Tailscale overlay mesh (WireGuard) for all remote access.
**Consequences**:
- ✅ No public IP needed, works behind CGNAT
- ✅ Zero-config mesh, automatic key rotation
- ✅ DERP relays ensure connectivity
- ✅ ACLs for access control
- ⚠️ Dependency on Tailscale control plane
- ⚠️ ~80ms latency via DERP relay (Mumbai)
- ⚠️ Not a full VPN replacement (no split tunneling config)

## ADR-002: SSH ProxyJump via Proxmox for OpenWrt Access
**Date**: 2026-11-12
**Status**: Accepted
**Context**: OpenWrt on LAN (192.168.1.1), no direct Tailscale, no public IP.
**Decision**: Use Proxmox as SSH bastion/jump host via ProxyJump.
**Consequences**:
- ✅ Single Tailscale endpoint (Proxmox) to manage
- ✅ OpenWrt never exposed to overlay directly
- ✅ Standard SSH feature, no extra software
- ⚠️ Proxmox must be up for OpenWrt access
- ⚠️ Two-hop latency (negligible for mgmt)
- ⚠️ Requires `AllowTcpForwarding yes` on Proxmox

## ADR-003: OpenWrt as Primary Gateway (Not Proxmox)
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Need router for Wi-Fi upstream (WWAN), VLANs, DHCP, DNS, firewall.
**Decision**: Use OpenWrt on TP-Link Archer C20 v5 as edge router/gateway.
**Consequences**:
- ✅ Purpose-built for routing/wireless
- ✅ Low power, always-on
- ✅ VLAN support via mt7530 switch
- ⚠️ Limited flash (16MB) → overlay space constrained
- ⚠️ No opkg/Python → Ansible raw/uci only
- ⚠️ Single 5GHz radio → STA/AP conflict
- ⚠️ End-of-life hardware (Archer C20 v5)

## ADR-004: Proxmox as Hypervisor (Not VM on Main Machine)
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Need always-on server for services, separate from control machine.
**Decision**: Dedicated Dell i3-M380 running Proxmox VE.
**Consequences**:
- ✅ Always available, independent of laptop
- ✅ Type-1 hypervisor, better performance
- ✅ ZFS/LVM-thin, snapshots, backups
- ✅ Hardware pass-through capable
- ⚠️ Older CPU (i3-M380, no AES-NI)
- ⚠️ 3.6 GB RAM limits VM count
- ⚠️ Single NIC (USB-Ethernet dongle)

## ADR-005: WWAN via 5GHz Wi-Fi Client (Not Ethernet WAN)
**Date**: 2026-11-12
**Status**: Accepted
**Context**: College dorm only provides Wi-Fi, no Ethernet port.
**Decision**: Use OpenWrt 5GHz radio (radio1) as STA to college Wi-Fi.
**Consequences**:
- ✅ Works with existing infrastructure
- ✅ VHT40 40MHz channel for decent throughput
- ⚠️ Shares radio with potential 5GHz AP (conflict)
- ⚠️ Dependent on college Wi-Fi stability
- ⚠️ Double NAT (college → OpenWrt → LAN)
- ⚠️ No IPv6 prefix delegation from upstream

## ADR-006: ULA IPv6 Only (No Global IPv6)
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Upstream (college) doesn't provide IPv6 prefix delegation.
**Decision**: Use OpenWrt-generated ULA (fd93:f7f5:2744::/48) for local IPv6.
**Consequences**:
- ✅ Local IPv6 for future services
- ✅ SLAAC + DHCPv6 works on LAN
- ⚠️ No internet IPv6 connectivity
- ⚠️ Must configure ULA-aware services

## ADR-007: Ansible for Configuration Management
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Need reproducible, version-controlled infrastructure config.
**Decision**: Ansible with `community.openwrt` for OpenWrt, native modules for Proxmox.
**Consequences**:
- ✅ Declarative, idempotent, auditable
- ✅ Vault for secrets
- ✅ Works over ProxyJump
- ⚠️ OpenWrt requires raw/uci modules (no Python)
- ⚠️ Learning curve for community.openwrt
- ⚠️ No drift detection (manual or scheduled runs)

## ADR-008: Self-Signed TLS for Internal Services
**Date**: 2026-11-12
**Status**: Accepted
**Context**: LuCI and Proxmox Web UI need HTTPS but no public CA for .lan domains.
**Decision**: Use self-signed certificates (EC P-256, 397-day validity).
**Consequences**:
- ✅ Encrypted management traffic
- ✅ No cost, no external dependency
- ⚠️ Browser warnings on first connect
- ⚠️ Must verify fingerprints manually
- ⚠️ Not suitable for public services

## ADR-009: VLAN Segmentation (VLAN 1 LAN, VLAN 2 WAN)
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Archer C20 v5 has mt7530 switch with 5 ports + CPU.
**Decision**: VLAN 1 (ports 1-4) for LAN, VLAN 2 (port 0) for WAN.
**Consequences**:
- ✅ Clear separation, standard OpenWrt pattern
- ✅ Easy to add more VLANs later (guest, IoT, mgmt)
- ⚠️ WAN port unused (WWAN via Wi-Fi)
- ⚠️ Single physical LAN port used (Proxmox on port 1)

## ADR-010: No IPv4 Port Forwarding / DMZ
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Double NAT (college → OpenWrt) makes port forwarding complex.
**Decision**: No inbound port forwarding; all remote access via Tailscale.
**Consequences**:
- ✅ Simplified firewall, no exposure
- ✅ Tailscale provides authenticated access
- ⚠️ Cannot host public services
- ⚠️ Requires Tailscale client for access

## ADR-011: Minimal Proxmox Firewall (Default-Deny)
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Proxmox exposed on LAN and Tailscale; need least privilege.
**Decision**: pve-firewall with `policy_in: DROP`, explicit allow for mgmt ports from LAN + Tailscale IP.
**Consequences**:
- ✅ Defense in depth
- ✅ Clear audit trail
- ⚠️ Must update rules for new services
- ⚠️ Tailscale `ts-input` chain bypasses pve-firewall

## ADR-012: rpcbind Removed (No NFS)
**Date**: 2026-11-12
**Status**: Accepted
**Context**: rpcbind (portmapper) on port 111 is unnecessary attack surface.
**Decision**: Stop, disable, mask rpcbind service.
**Consequences**:
- ✅ Eliminates port 111 exposure
- ✅ No NFS use case in this lab
- ⚠️ Cannot mount NFS shares (not needed)

---

## Hardware Decisions (Future)

### HWD-001: Router Replacement Target
**Target**: Dual-radio Wi-Fi 6/6E router with ≥128MB RAM, ≥16MB flash, OpenWrt support.
**Candidates**: 
- GL.iNet GL-MT6000 (Flint 2) - MediaTek Filogic 820, 1GB RAM, 8GB eMMC
- Banana Pi BPI-R3 - MediaTek Filogic 830, 2GB RAM, 8GB eMMC
- NanoPi R5S/R6S - Rockchip RK3568, 4GB RAM, eMMC

**Requirements**:
- Two independent 5GHz radios (or 6GHz) → STA + AP simultaneously
- ≥128MB RAM for packages
- ≥16MB storage for overlay
- OpenWrt 23.05+ support
- GbE ports (2.5G preferred)

### HWD-002: Proxmox Upgrade Path
**Target**: Mini PC with Intel N100/N200 or AMD Ryzen 5000, 16-32GB RAM, NVMe.
**Timeline**: When VM workloads exceed current capacity.

---

*See [Security Decisions](security-decisions.md) for security-specific ADRs, [Phase 4 Report](../phases/phase-4.md) for implementation.*