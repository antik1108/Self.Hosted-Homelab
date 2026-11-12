# Network Design Document

## Overview

This document describes the network architecture for the homelab, including IP addressing, VLANs, routing, and security zones.

## IP Address Plan

### Main Network (LAN)

| Device | Hostname | IP Address | MAC Address | Notes |
|--------|----------|------------|-------------|-------|
| Router | main-router | 192.168.1.1 | - | OpenWrt (Archer C20 v5) |
| Proxmox | pve-host | 192.168.1.10 | aa:bb:cc:dd:ee:ff | Management IP |
| MacBook | macbook | 192.168.1.50 | 11:22:33:44:55:66 | Control machine |
| DHCP Pool | - | 192.168.1.100-250 | - | Dynamic clients |

### Subnet Details

- **Network**: 192.168.1.0/24
- **Gateway**: 192.168.1.1
- **Netmask**: 255.255.255.0
- **DNS**: 1.1.1.1, 8.8.8.8, 9.9.9.9
- **DHCP Range**: 192.168.1.100 - 192.168.1.250
- **Lease Time**: 12 hours

### Reserved IPs (Static DHCP)

| IP | Device | Purpose |
|----|--------|---------|
| 192.168.1.1 | main-router | Router management |
| 192.168.1.10 | pve-host | Proxmox management |
| 192.168.1.50 | macbook | Control machine |
| 192.168.1.20-30 | Future servers | Reserved for VMs/containers |
| 192.168.1.30-40 | Future NAS | Reserved for storage |
| 192.168.1.40-49 | Future APs | Reserved for access points |

## Network Topology

```
Internet (WAN)
     │
     ▼
┌─────────────────────────────────────┐
│         TP-Link Archer C20 v5       │
│            (OpenWrt)                │
│                                     │
│  WAN Port ◄── ISP Connection        │
│  LAN Ports ──► Switch (4 ports)     │
│  WiFi 2.4GHz ──► radio0             │
│  WiFi 5GHz ──► radio1               │
└─────────────────────────────────────┘
     │
     ├──► Port 1: Proxmox Host (192.168.1.10)
     ├──► Port 2: Available
     ├──► Port 3: Available
     └──► Port 4: Available
```

## VLAN Plan (Future)

| VLAN ID | Name | Subnet | Purpose |
|---------|------|--------|---------|
| 1 | management | 192.168.1.0/24 | Infrastructure management |
| 10 | servers | 192.168.10.0/24 | VMs and containers |
| 20 | iot | 192.168.20.0/24 | IoT devices (isolated) |
| 30 | guest | 192.168.30.0/24 | Guest WiFi (isolated) |
| 40 | storage | 192.168.40.0/24 | NAS/backup traffic |

## Firewall Zones

### LAN Zone
- **Policy**: Accept all
- **Interfaces**: lan, wifi
- **Forward**: Allow to WAN

### WAN Zone
- **Policy**: Reject input, Accept output, Reject forward
- **Masquerading**: Enabled
- **MSS Clamping**: Enabled
- **Interfaces**: wan, wan6

### Forwarding Rules
- LAN → WAN: Allow (internet access)
- WAN → LAN: Block (except established/related)
- Port forwards: Defined per service

## DNS Architecture

```
Clients (LAN)
     │
     ▼
┌─────────────┐
│  dnsmasq    │ ◄── Local DNS (DHCP hostnames)
│  (OpenWrt)  │
└─────────────┘
     │
     ├──► 1.1.1.1 (Cloudflare)
     ├──► 8.8.8.8 (Google)
     └──► 9.9.9.9 (Quad9)
```

- Local domain: `.lan`
- DNSSEC: Enabled
- Rebind protection: Enabled

## DHCP Configuration

- **Server**: dnsmasq on OpenWrt
- **Range**: 192.168.1.100-250
- **Static leases**: MAC-based reservations for infrastructure
- **Options**: Gateway, DNS, Domain, NTP
- **IPv6**: SLAAC + DHCPv6

## Wireless Configuration

### 2.4GHz (radio0)
- **SSID**: HomeLab-2G
- **Channel**: Auto
- **Width**: HT40 (40MHz)
- **Encryption**: WPA2-PSK (CCMP)
- **Country**: IN (India)

### 5GHz (radio1)
- **SSID**: HomeLab-5G
- **Channel**: Auto
- **Width**: VHT80 (80MHz)
- **Encryption**: WPA3-SAE / WPA2-PSK (mixed)
- **Country**: IN (India)
- **802.11w**: Required (PMF)

## Proxmox Network

### Bridges
| Bridge | Ports | IP | Purpose |
|--------|-------|----|---------|
| vmbr0 | eno1 | 192.168.1.10/24 | Management + VMs (LAN) |
| vmbr1 | eno2 | - | Isolated VM network (future) |

### VM Network Model
- All VMs connect to vmbr0 by default
- VLAN tagging for future segmentation
- VirtIO drivers for performance

## Security Considerations

1. **SSH**: Key-only authentication, non-standard port (future)
2. **Web UI**: HTTPS only, certificate management
3. **WiFi**: WPA3 where supported, strong passwords
4. **Firewall**: Default deny, explicit allow
5. **Updates**: Regular security patches
6. **Backups**: Encrypted, off-site storage

## Monitoring (Future)

- Prometheus + Grafana for metrics
- SNMP for network devices
- Log aggregation (Loki/ELK)
- Alerting via email/webhook

## Change History

| Date | Change | Author |
|------|--------|--------|
| 2026-11-12 | Initial design | - |