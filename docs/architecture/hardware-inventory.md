# Hardware Inventory

## OpenWrt Router: TP-Link Archer C20 v5

### System Information
| Property | Value |
|----------|-------|
| **Model** | TP-Link Archer C20 v5 |
| **Board Name** | tplink,archer-c20-v5 |
| **CPU** | MediaTek MT7628AN ver:1 eco:2 (MIPS 24KEc, 580 MHz) |
| **Architecture** | MIPS 24KEc (mipsel_24kc) |
| **RAM** | 64 MB (57,980 KB total, ~16 MB free typical) |
| **Flash** | 16 MB (SPI) |
| **Rootfs Type** | squashfs (read-only) + JFFS2 overlay |
| **Overlay Size** | 960 KB (93% used, ~72 KB free) |

### Firmware
| Property | Value |
|----------|-------|
| **OpenWrt Version** | 25.12.6 |
| **Build ID** | r33311-981c978eff |
| **Kernel** | 6.12.112 |
| **Target** | ramips/mt76x8 |
| **LuCI Version** | 26.312.33065 |
| **Build Date** | 2026-11-10 (timestamp 1792273538) |

### Wireless Radios

#### Radio 0: 2.4 GHz (MT7628AN integrated)
| Property | Value |
|----------|-------|
| **Phy** | phy0 (platform/10300000.wmac) |
| **Driver** | mt76x8 |
| **Band** | 2.4 GHz |
| **Channels** | 1-13 (2412-2472 MHz) |
| **HT Modes** | HT20, HT40 |
| **TX Power** | Up to 20 dBm |
| **Current Config** | CH1, HT20, 20 dBm |
| **AP Interface** | phy0-ap0 (master, bridged to br-lan) |
| **SSID** | OpenWrt (configured), HomeLab-2G (Ansible target) |
| **Encryption** | WPA2-PSK (psk2) / WPA2+CCMP (Ansible target) |

#### Radio 1: 5 GHz (MT76x2 PCIe)
| Property | Value |
|----------|-------|
| **Phy** | phy1 (pci0000:00/0000:00:00.0/0000:01:00.0) |
| **Driver** | mt76x2 |
| **Band** | 5 GHz |
| **Channels** | 36-165 (5180-5825 MHz) |
| **HT/VHT Modes** | HT20, HT40, VHT20, VHT40, VHT80 |
| **TX Power** | Up to 16 dBm (varies by channel) |
| **Interface Combinations** | `#channels <= 1, STA/AP BI must match` |
| **Current STA Config** | CH149, VHT40, 13 dBm (Rishihood_Learner) |
| **Current AP Config** | CH60, VHT40, 10 dBm, **NO ENCRYPTION, DISABLED** |
| **STA Interface** | phy1-sta0 (managed, WWAN client) |
| **AP Interface** | phy1-ap0 (master, DISABLED) |

### Switch (mt7530)
| Property | Value |
|----------|-------|
| **Chip** | MediaTek MT7530 (5-port Gigabit switch with VLAN) |
| **CPU Port** | Port 6 (tagged for VLAN 1 & 2) |
| **VLAN 1 (LAN)** | Ports 1, 2, 3, 4, 6t → eth0.1 → br-lan |
| **VLAN 2 (WAN)** | Port 0, 6t → eth0.2 → wan interface |

### Interfaces
| Interface | Device | Type | MAC Address | Status |
|-----------|--------|------|-------------|--------|
| br-lan | bridge | static | e8:48:b8:14:9f:f0 | UP |
| eth0.1 | vlan | bridge port | e8:48:b8:14:9f:f0 | UP |
| eth0.2 | vlan | dhcp | e8:48:b8:14:9f:f1 | DOWN |
| phy0-ap0 | wifi | ap | e8:48:b8:14:9f:f0 | UP |
| phy1-ap0 | wifi | ap | e8:48:b8:14:9f:ef | DOWN |
| phy1-sta0 | wifi | sta | ea:48:b8:14:9f:ef | UP |
| lo | loopback | static | - | UP |

### Storage Constraints
| Metric | Value | Impact |
|--------|-------|--------|
| Overlay Total | 960 KB | Fixed by flash partition |
| Overlay Used | ~888 KB (93%) | Cannot install packages |
| Overlay Free | ~72 KB | Only UCI commits, small files |
| ROM Available | ~8 MB (squashfs) | Read-only, contains base system |
| **No opkg** | Package manager missing | Cannot install python3, tools |

### Known Limitations
1. **Overlay near capacity** - JFFS2 needs free space for GC; 72 KB is minimal
2. **No python3** - Ansible `ping` module fails; must use `raw` or `community.openwrt.uci`
3. **Single 5GHz channel** - STA and AP must share channel (beacon interval match)
4. **No USB storage** - USB 2.0 port exists but no overlay expansion configured
5. **Limited RAM** - 64 MB constrains concurrent services

---

## Proxmox Host: Dell Inspiron N4010

### System Information
| Property | Value |
|----------|-------|
| **Model** | Dell Inspiron N4010 |
| **CPU** | Intel Core i3-M380 @ 2.53 GHz (2 cores, 4 threads) |
| **Virtualization** | VT-x supported |
| **RAM** | 4 GB DDR3 (3.6 GiB usable, ~1.9 GiB free) |
| **Storage** | 238 GB SSD + 465 GB HDD |

### Firmware
| Property | Value |
|----------|-------|
| **Proxmox VE** | 9.2.2 (pve-manager 9.2.2) |
| **Kernel** | 7.0.2-6-pve |
| **Base OS** | Debian 13 (trixie) |
| **Hostname** | homelab |

### Storage Layout
| Device | Size | Partition | Filesystem | Use |
|--------|------|-----------|------------|-----|
| sda (SSD) | 238.5 GB | sda1 (1 MB) | BIOS boot | - |
| | | sda2 (1 GB) | ext4 | /boot |
| | | sda3 (237.5 GB) | LVM2 (pve VG) | - |
| └─ pve-swap | 4 GB | swap | - | Swap |
| └─ pve-root | 70.4 GB | ext4 | / | Root (10% used) |
| └─ pve-data | 144.2 GB | LVM-thin | - | VM images (0% used) |
| sdb (HDD) | 465.8 GB | sdb1 (479 GB) | ext4 | /mnt/pve/hdd-storage |

### Proxmox Storage
| Storage ID | Type | Total | Used | Available | Content |
|------------|------|-------|------|-----------|---------|
| local | dir | 72 GB | 6.8 GB | 61.5 GB | vztmpl, iso, backup |
| local-lvm | lvmthin | 151 GB | 0 | 151 GB | rootdir, images |
| hdd-storage | dir | 479 GB | 2 MB | 455 GB | vztmpl, iso, backup |

### Network Interfaces
| Interface | Type | IP Address | Bridge Ports | Status |
|-----------|------|------------|--------------|--------|
| lo | loopback | 127.0.0.1/8, ::1/128 | - | UP |
| enx00e04c685ea8 | physical (USB-Ethernet) | 192.168.1.10/24 | vmbr0 | UP |
| wlp3s0 | wifi (internal) | - | - | DOWN |
| vmbr0 | bridge | 192.168.1.10/24 | enx00e04c685ea8 | UP |
| tailscale0 | p2p (Tailscale) | 100.122.224.81/32, fd7a:115c:a1e0::de28:e052/128 | - | UP |

### MAC Addresses
| Interface | MAC Address |
|-----------|-------------|
| enx00e04c685ea8 | 00:e0:4c:68:5e:a8 |
| vmbr0 | 00:e0:4c:68:5e:a8 (inherited) |

### Services Running
| Service | Port | Bind Address | Notes |
|---------|------|--------------|-------|
| sshd | 22 | 0.0.0.0, :: | Key + password auth |
| pveproxy | 8006 | *:* | Web UI (firewall restricted) |
| spiceproxy | 3128 | *:* | VM console (firewall restricted) |
| pvedaemon | 85 | 127.0.0.1 | Local only |
| postfix | 25 | 127.0.0.1, ::1 | Local only |
| tailscaled | 41641 | 0.0.0.0, :: | Tailscale |
| chronyd | 323 | 127.0.0.1, ::1 | NTP |

### Resource Utilization (Typical)
| Resource | Usage | Available |
|----------|-------|-----------|
| CPU | ~5% idle | 4 threads |
| RAM | ~1.7 GiB | 1.9 GiB free |
| Swap | ~254 MiB | 4 GiB |
| SSD Root | ~7 GB | 63 GB free |
| LVM-thin | 0 GB | 151 GB free |

---

## MacBook Pro (Control Machine)

### System Information
| Property | Value |
|----------|-------|
| **Model** | MacBook Pro (Apple Silicon) |
| **OS** | macOS (latest) |
| **Tailscale** | 1.78+ |
| **Ansible** | 2.21.6 (venv) |
| **Python** | 3.13+ |

### Network Interfaces
| Interface | Type | MAC Address | Role |
|-----------|------|-------------|------|
| en0 | WiFi | 34:66:91:ef:4e:2d | Tailscale (100.110.79.92) |
| en3 | Ethernet (USB) | 1a:f7:84:8b:d8:dd | Local network |
| en4 | Ethernet (USB) | 1a:f7:84:8b:d8:de | Local network |
| en1 | Thunderbolt 1 | 36:88:ca:65:ab:40 | Local |
| en2 | Thunderbolt 2 | 36:88:ca:65:ab:44 | Local |

### SSH Keys
| Key File | Type | Fingerprint | Deployed To |
|----------|------|-------------|-------------|
| ~/.ssh/id_ed25519 | Ed25519 | itsantikmondal@gmail.com | Proxmox (authorized_keys) |
| ~/.ssh/id_rsa | RSA | antikmondal@Antiks-MacBook-Pro.local | OpenWrt (authorized_keys) |

---

## Hardware Compatibility Matrix for Migration

| Feature | Archer C20 v5 | Considerations for Replacement |
|---------|---------------|--------------------------------|
| **Flash** | 16 MB | Need ≥16 MB for OpenWrt + packages |
| **RAM** | 64 MB | Need ≥128 MB for comfortable operation |
| **2.4GHz Radio** | MT7628AN (integrated) | Most mt76xx devices supported |
| **5GHz Radio** | MT76x2 (PCIe) | Check mt76x2/mt7915/mt7921 support |
| **Switch** | MT7530 (5-port GbE) | VLAN support varies by switch chip |
| **USB** | 1× USB 2.0 | For storage/3G; check OTG support |
| **PoE** | No | Not required |
| **OpenWrt Support** | Full (ramips/mt76x8) | Check target device support status |
| **Overlay Size** | 960 KB (tight) | Prefer devices with larger overlay or USB expansion |

---

*See [Architecture Overview](overview.md) for system context, [Network Topology](network-topology.md) for connections, [IP Addressing](ip-addressing.md) for allocation, and [Security Architecture](security-architecture.md) for controls.*