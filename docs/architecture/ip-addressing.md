# IP Addressing Plan

## Network Summary

| Network | CIDR | Gateway | VLAN | Purpose |
|---------|------|---------|------|---------|
| **LAN** | 192.168.1.0/24 | 192.168.1.1 | 1 | Infrastructure management, Proxmox, DHCP clients |
| **WWAN** | 10.7.0.0/19 | 10.7.0.1 | - | Upstream internet via college Wi-Fi |
| **Tailscale** | 100.64.0.0/10 | - | - | Overlay mesh (managed by Tailscale) |
| **ULA (IPv6)** | fd93:f7f5:2744::/48 | fd93:f7f5:2744::1 | 1 | Local IPv6 (OpenWrt ULA) |

---

## LAN (192.168.1.0/24) - Detailed Allocation

### Static Assignments (Infrastructure)

| IP Address | Device | Hostname | MAC Address | Method | Notes |
|------------|--------|----------|-------------|--------|-------|
| **192.168.1.1** | OpenWrt Router | main-router | e8:48:b8:14:9f:f0 | Static | LAN gateway, management |
| **192.168.1.10** | Proxmox Host | pve-host | 00:e0:4c:68:5e:a8 | Static (host) + DHCP reservation | Management, Tailscale endpoint |
| **192.168.1.50** | MacBook Pro | macbook | 34:66:91:ef:4e:2d | DHCP reservation | Control machine (when on LAN) |

### DHCP Pool

| Range | Count | Lease Time | Purpose |
|-------|-------|------------|---------|
| **192.168.1.100 - 192.168.1.250** | 151 | 12 hours | Dynamic clients (phones, laptops, IoT) |

### Reserved for Future Use (Not in DHCP Pool)

| IP Range | Purpose | Notes |
|----------|---------|-------|
| 192.168.1.2 - 192.168.1.9 | Infrastructure expansion | Additional routers, APs, switches |
| 192.168.1.11 - 192.168.1.19 | Proxmox VMs/CTs | Static or DHCP reservations |
| 192.168.1.20 - 192.168.1.30 | Future servers | NAS, media, automation |
| 192.168.1.31 - 192.168.1.40 | Storage | NAS, backup targets |
| 192.168.1.41 - 192.168.1.49 | Access Points | Wireless APs |
| 192.168.1.51 - 192.168.1.99 | Buffer | Unallocated |

### DHCP Options (dnsmasq)

| Option | Value |
|--------|-------|
| **Router (Gateway)** | 192.168.1.1 |
| **DNS Servers** | 192.168.1.1 (local), 1.1.1.1, 8.8.8.8, 9.9.9.9 |
| **Domain** | lan |
| **NTP Server** | 192.168.1.1 (OpenWrt) |
| **Lease Time** | 12 hours (43200 seconds) |

---

## WWAN (10.7.0.0/19) - Upstream Network

> **Note**: The college network topology is not fully under our control. Values below are observed from DHCP lease.

### Observed Configuration

| Parameter | Value | Source |
|-----------|-------|--------|
| **Network** | 10.7.0.0/19 | DHCP lease |
| **OpenWrt WWAN IP** | 10.7.28.50 | DHCP lease (phy1-sta0) |
| **Subnet Mask** | 255.255.224.0 (/19) | DHCP lease |
| **Upstream Gateway** | 10.7.0.1 | Route table |
| **DHCP Server** | 10.1.0.2 | DHCP lease |
| **DNS Servers** | 8.8.8.8, 10.5.7.1 | DHCP lease / resolv.conf |
| **Lease Time** | 18 hours (64800s) | DHCP lease |
| **SSID** | Rishihood_Learner | Wireless config |
| **Channel** | 149 (5745 MHz) | Wireless config |
| **Width** | VHT40 (40 MHz) | Wireless config |
| **Encryption** | WPA2-PSK | Wireless config |

### Routing (OpenWrt)
```
default via 10.7.0.1 dev phy1-sta0 src 10.7.28.50
10.7.0.0/19 dev phy1-sta0 scope link src 10.7.28.50
192.168.1.0/24 dev br-lan scope link src 192.168.1.1
```

> **Unresolved**: Earlier inspection showed gateway as 10.7.0.1; DHCP server as 10.1.0.2. The /19 (10.7.0.0/19) encompasses both. Current live route confirms 10.7.0.1 as default gateway.

---

## Tailscale Overlay (100.64.0.0/10)

Managed entirely by Tailscale. We only observe endpoints.

### Known Endpoints

| Device | Tailscale IP | Hostname | Status | Relay |
|--------|--------------|----------|--------|-------|
| Proxmox | 100.122.224.81 | homelab | Active | - |
| MacBook | 100.110.79.92 | antiks-macbook-pro | Active | DERP "bom" |
| iPad Gen 8 | 100.123.25.117 | ipad-gen-8 | Offline | - |
| Realme C31 | 100.101.205.84 | realme-c31 | Offline (16h) | - |

### Tailscale Ports (Proxmox)
| Port | Protocol | Purpose | Binding |
|------|----------|---------|---------|
| 41641 | UDP | WireGuard data / DERP | 0.0.0.0, :: |
| 42522 | TCP | Control plane (IPv4) | tailscale0 (100.122.224.81) |
| 36222 | TCP | Control plane (IPv6) | tailscale0 (fd7a:115c:a1e0::de28:e052) |

### Traffic Flow
```
MacBook (100.110.79.92)
    │
    ├─► DERP Relay (bom, Mumbai) ──► Proxmox (100.122.224.81)
    │       │
    │       └─► UDP 41641 (WireGuard encapsulated)
    │
    └─► Direct connection not established (NAT/firewall)
```

---

## IPv6 (ULA - Unique Local Address)

### OpenWrt ULA Configuration
| Parameter | Value |
|-----------|-------|
| **ULA Prefix** | fd93:f7f5:2744::/48 |
| **LAN Delegation** | fd93:f7f5:2744::/60 (to br-lan) |
| **Router Address** | fd93:f7f5:2744::1/60 |
| **Assignment** | SLAAC + DHCPv6 (managed-config, other-config) |

### IPv6 Addresses (OpenWrt)
| Interface | Address | Scope |
|-----------|---------|-------|
| br-lan | fd93:f7f5:2744::1/60 | Global ULA |
| br-lan | fe80::e848:b8ff:fe14:9fef/64 | Link-local |
| phy1-sta0 | fe80::ea48:b8ff:fe14:9ff1/64 | Link-local |
| phy1-ap0 | fe80::ea48:b8ff:fe14:9ff0/64 | Link-local |
| lo | ::1/128 | Loopback |

### Proxmox IPv6
| Interface | Address | Scope |
|-----------|---------|-------|
| vmbr0 | fd93:f7f5:2744::/64 (SLAAC) | Global ULA |
| tailscale0 | fd7a:115c:a1e0::de28:e052/128 | Tailscale ULA |
| lo | ::1/128 | Loopback |

> **Note**: No IPv6 prefix delegation from upstream (college network). ULA is locally generated only.

---

## DNS Architecture

### OpenWrt (dnsmasq) - Authoritative for .lan

| Zone | Type | Records |
|------|------|---------|
| **lan** | Forward | main-router (192.168.1.1), pve-host (192.168.1.10), macbook (192.168.1.50) + DHCP leases |
| **Reverse (1.168.192.in-addr.arpa)** | PTR | Auto-generated from DHCP leases |

### Upstream DNS (Forwarders)
| Server | IP | Purpose |
|--------|----|---------|
| Cloudflare | 1.1.1.1 | Primary (DoH capable) |
| Google | 8.8.8.8 | Secondary |
| Quad9 | 9.9.9.9 | Tertiary (security-focused) |
| College DNS | 10.5.7.1 | From WWAN DHCP (fallback) |

### DNS Security
| Feature | Status |
|---------|--------|
| DNSSEC Validation | Enabled |
| Rebind Protection | Enabled |
| Local Service Only | Enabled (but currently listens on all interfaces) |
| Cache Size | 1000 entries |
| EDNS Packet Max | 1232 bytes |

---

## NTP Configuration

### OpenWrt (chrony client)
| Server | Pool | Notes |
|--------|------|-------|
| 0.openwrt.pool.ntp.org | pool.ntp.org | Primary |
| 1.openwrt.pool.ntp.org | pool.ntp.org | Secondary |
| 2.openwrt.pool.ntp.org | pool.ntp.org | Tertiary |
| 3.openwrt.pool.ntp.org | pool.ntp.org | Quaternary |

### Proxmox (chronyd)
| Source | Stratum | Status |
|--------|---------|--------|
| time.cloudflare.com | 3 | Active |
| ntp1.ggsrv.de | 2 | Active |
| ntp7.mum-in.hosts.301-mo | 2 | Active (selected) |
| ec2-13-126-27-131.ap-south-1 | 2 | Active (selected *) |

**System Time**: Synchronized, offset ~0.2ms, stratum 3

---

## Firewall Management Ports

### OpenWrt (Management on LAN Only)
| Service | Port | Protocol | Bind Address | Access |
|---------|------|----------|--------------|--------|
| SSH (dropbear) | 22 | TCP | 192.168.1.1 | LAN only |
| HTTPS (LuCI) | 443 | TCP | 192.168.1.1 | LAN only |
| HTTP (redirect) | 80 | TCP | 192.168.1.1 | LAN only → 307 to HTTPS |

### Proxmox (Management on LAN + Tailscale via Firewall)
| Service | Port | Protocol | Bind Address | Firewall Allow |
|---------|------|----------|--------------|----------------|
| SSH (sshd) | 22 | TCP | 0.0.0.0, :: | 192.168.1.0/24, 100.122.224.81/32 |
| Web UI (pveproxy) | 8006 | TCP | *:* | 192.168.1.0/24, 100.122.224.81/32 |
| SPICE Proxy | 3128 | TCP | *:* | 192.168.1.0/24, 100.122.224.81/32 |
| Tailscale UDP | 41641 | UDP | 0.0.0.0, :: | Any (DERP relay) |
| Tailscale TCP | 42522, 36222 | TCP | tailscale0 | Tailscale subnet only |

### Tailscale (ts-input chain - handled by tailscaled)
| Service | Port | Protocol | Interface | Action |
|---------|------|----------|-----------|--------|
| All traffic | * | * | tailscale0 | ACCEPT |
| WireGuard/DERP | 41641 | UDP | * | ACCEPT |
| Tailscale subnet | 100.64.0.0/10 | * | !tailscale0 | DROP |

---

## IP Address Quick Reference

| CIDR | Name | Gateway | DHCP | Key Hosts |
|------|------|---------|------|-----------|
| 192.168.1.0/24 | LAN | 192.168.1.1 | 100-250 | .1 (OWrt), .10 (PVE), .50 (MB) |
| 10.7.0.0/19 | WWAN | 10.7.0.1 | DHCP | .28.50 (OWrt WWAN) |
| 100.64.0.0/10 | Tailscale | - | - | .122.224.81 (PVE), .110.79.92 (MB) |
| fd93:f7f5:2744::/48 | ULA IPv6 | fd93:f7f5:2744::1 | SLAAC | ::1 (OWrt), ::/64 (PVE) |

---

*See [Architecture Overview](overview.md) for context, [Network Topology](network-topology.md) for diagrams, [Hardware Inventory](hardware-inventory.md) for device details, and [Security Architecture](security-architecture.md) for controls.*