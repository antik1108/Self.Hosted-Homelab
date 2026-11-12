# Architecture Overview

## System Context

This homelab is a **two-node infrastructure** designed for remote management, service hosting, and network experimentation. It runs behind a university Wi-Fi network with no public IP, using Tailscale for secure overlay connectivity.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL CONTEXT                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐              │
│  │  College Wi-Fi  │    │   Tailscale     │    │   Internet      │              │
│  │  (Rishihood_    │    │   Control       │    │   (via WWAN)    │              │
│  │   Learner)      │    │   Plane         │    │                 │              │
│  │  10.7.0.0/19    │    │   (DERP/Relay)  │    │                 │              │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘              │
│           │                      │                      │                       │
└───────────┼──────────────────────┼──────────────────────┼───────────────────────┘
            │                      │                      │
            ▼                      ▼                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              HOMELAB INFRASTRUCTURE                               │
│                                                                                   │
│  ┌─────────────────────────────────────┐    ┌─────────────────────────────────┐  │
│  │      OPENWRT ROUTER                 │    │      PROXMOX VE HOST            │  │
│  │  TP-Link Archer C20 v5              │    │  Dell Inspiron N4010            │  │
│  │  OpenWrt 25.12.6                    │    │  Proxmox VE 9.2.2               │  │
│  │                                     │    │                                 │  │
│  │  Interfaces:                        │    │  Interfaces:                    │  │
│  │  • br-lan: 192.168.1.1/24 (LAN)    │◄───│  • vmbr0: 192.168.1.10/24       │  │
│  │  • phy1-sta0: 10.7.28.50/19 (WWAN) │    │  • tailscale0: 100.122.224.81   │  │
│  │  • phy0-ap0: 2.4GHz AP              │    │  • enx00e04c685ea8 (physical)   │  │
│  │  • phy1-sta0: 5GHz STA (upstream)   │    │                                 │  │
│  │  • phy1-ap0: 5GHz AP (DISABLED)     │    │  Storage:                       │  │
│  │                                     │    │  • local-lvm: 151GB (LVM-thin)  │  │
│  │  Services:                          │    │  • local: 72GB (dir)            │  │
│  │  • dnsmasq (DHCP/DNS)               │    │  • hdd-storage: 479GB (dir)     │  │
│  │  • dropbear (SSH, LAN only)         │    │                                 │  │
│  │  • uhttpd (LuCI HTTPS, LAN only)    │    │  VMs/CTs: None (ready)          │  │
│  │  • firewall (nftables/fw4)          │    │                                 │  │
│  └─────────────────────────────────────┘    └─────────────────────────────────┘  │
│           ▲                                              ▲                        │
│           │ Ethernet (192.168.1.0/24)                    │ Tailscale             │
│           └──────────────────────────────────────────────┘                        │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
            ▲
            │ Tailscale (100.x.x.x)
            ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              CONTROL MACHINE                                      │
│  MacBook Pro (macOS)                                                             │
│  Tailscale IP: 100.110.79.92                                                    │
│  Ansible Controller                                                              │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Primary Role | Secondary Roles |
|-----------|--------------|-----------------|
| **OpenWrt Router** | Gateway, NAT, DHCP, DNS, WiFi AP, WWAN client | Firewall, NTP client, SSH jump target |
| **Proxmox Host** | Hypervisor (VMs/LXC), Storage | Tailscale exit node, SSH jump host for OpenWrt |
| **MacBook (Control)** | Ansible controller, Tailscale client | Operator workstation |

## Data Flows

### Management Traffic
```
MacBook (100.110.79.92)
    │
    ├─► Tailscale ──► Proxmox (100.122.224.81):22 (SSH)
    │                       │
    │                       └─► ProxyJump ──► OpenWrt (192.168.1.1):22 (SSH)
    │
    ├─► Tailscale ──► Proxmox (100.122.224.81):8006 (HTTPS Web UI)
    │
    └─► Tailscale ──► Proxmox (100.122.224.81):3128 (SPICE proxy)
```

### Client Traffic (LAN Devices)
```
LAN Device (192.168.1.x)
    │
    ├─► OpenWrt br-lan (192.168.1.1) ──► DNS (dnsmasq) ──► Upstream (8.8.8.8, 1.1.1.1)
    │
    ├─► OpenWrt br-lan ──► DHCP (dnsmasq) ──► Lease 192.168.1.100-250
    │
    └─► OpenWrt br-lan ──► NAT ──► phy1-sta0 (10.7.28.50) ──► College GW (10.7.0.1) ──► Internet
```

### Proxmox Outbound
```
Proxmox (192.168.1.10)
    │
    ├─► OpenWrt (192.168.1.1) ──► Internet (updates, NTP, Tailscale control)
    │
    └─► Tailscale (100.122.224.81) ──► DERP Relay (bom) ──► Tailscale Control Plane
```

## Trust Boundaries & Security Zones

| Zone | CIDR | Trust Level | Ingress Policy | Egress Policy |
|------|------|-------------|----------------|---------------|
| **WWAN (Upstream)** | 10.7.0.0/19 | **Untrusted** | REJECT all (except DHCP, ICMP) | ALLOW (NAT) |
| **LAN** | 192.168.1.0/24 | **Trusted** | ACCEPT (management) | ACCEPT |
| **Tailscale Overlay** | 100.64.0.0/10 | **Trusted** | ACCEPT (via ts-input) | ACCEPT |
| **Proxmox Management** | 100.122.224.81/32 | **Trusted** | ACCEPT (SSH, 8006, 3128) | ACCEPT |
| **OpenWrt Management** | 192.168.1.1/24 | **Trusted** | ACCEPT (SSH, HTTPS) | ACCEPT |

## Key Design Decisions

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| **WWAN as Wi-Fi client** | No Ethernet WAN available in dorm/college | Single radio must share channel for STA+AP; 5GHz AP disabled |
| **Tailscale for remote access** | No public IP, no port forwarding possible | Depends on Tailscale control plane; DERP relay adds latency |
| **ProxyJump via Proxmox** | OpenWrt not on Tailscale; Proxmox is always-on | Extra hop; Proxmox must be reachable |
| **Static IPs for infrastructure** | Predictable addresses for automation | Must manage DHCP pool to avoid conflicts |
| **Ansible Vault for secrets** | GitOps-friendly, encrypted at rest | Requires vault password at runtime |
| **Hardware abstraction in group_vars** | Enables router migration without playbook changes | More complex variable structure |

## Scalability Considerations

| Aspect | Current | Future Expansion |
|--------|---------|------------------|
| **IP Space** | 192.168.1.0/24 (254 hosts) | VLAN segmentation (10/20/30/40) when needed |
| **Proxmox Storage** | 151GB + 479GB | Add NFS/CEPH for HA |
| **WiFi** | 2.4GHz AP only | Replace router for 5GHz AP + better client support |
| **Remote Access** | Tailscale only | Consider WireGuard + Tailscale hybrid |
| **Automation** | Ansible only | Add Terraform for cloud resources |

## Failure Domains

| Failure | Impact | Detection | Recovery |
|---------|--------|-----------|----------|
| **College Wi-Fi down** | No internet, no Tailscale | Tailscale status "offline" | Wait for restore; local LAN still works |
| **OpenWrt crash** | No LAN, no DHCP/DNS, no internet | Proxmox can't reach 192.168.1.1 | Sysupgrade restore from backup |
| **Proxmox crash** | No VMs, no Tailscale, no jump host | Tailscale shows "offline" | Reinstall Proxmox, restore /etc/pve |
| **Tailscale control plane down** | No new connections; existing work | Tailscale status | Wait; local LAN access unaffected |
| **MacBook lost** | No Ansible control | N/A | Re-provision from repo + vault password |

---

*See [Network Topology](network-topology.md) for detailed diagrams, [Hardware Inventory](hardware-inventory.md) for specs, [IP Addressing](ip-addressing.md) for complete allocation, and [Security Architecture](security-architecture.md) for threat model and controls.*