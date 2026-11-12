# Homelab Infrastructure as Code

[![Ansible](https://img.shields.io/badge/Ansible-2.21%2B-EE0000?style=flat&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![OpenWrt](https://img.shields.io/badge/OpenWrt-25.12.6-46A5E6?style=flat&logo=openwrt&logoColor=white)](https://openwrt.org/)
[![Proxmox VE](https://img.shields.io/badge/Proxmox%20VE-9.2.2-E57000?style=flat&logo=proxmox&logoColor=white)](https://www.proxmox.com/)
[![Tailscale](https://img.shields.io/badge/Tailscale-1.78%2B-2D8CFF?style=flat&logo=tailscale&logoColor=white)](https://tailscale.com/)

This repository contains the Infrastructure as Code (IaC) configuration for a personal homelab, managed with **Ansible**. The homelab runs on commodity hardware behind a university Wi-Fi network, with secure remote access via Tailscale.

## Project Overview

### What I'm Building

A resilient, remotely manageable homelab that:
- Runs on a 2010 Dell Inspiron N4010 laptop (Proxmox VE) and a TP-Link Archer C20 v5 router (OpenWrt)
- Provides secure remote access from anywhere via Tailscale
- Uses Ansible for all configuration management
- Supports future router migration without re-engineering the entire stack

### Why This Exists

- **Learning**: Hands-on experience with infrastructure automation, network engineering, and security hardening
- **Portfolio**: Demonstrates real-world IaC, network design, and operational practices
- **Utility**: Actually runs services (future: media server, home automation, development environments)
- **Resilience**: Designed to survive hardware replacement and network changes

---

## Current Project Status

| Phase | Status | Description |
|-------|--------|-------------|
| **Phase 1** | ✅ Complete | Development environment, Ansible setup |
| **Phase 2** | ✅ Complete | Network documentation and audit |
| **Phase 3** | ✅ Complete | Backup strategy, SSH keys, Ansible Vault, overlay cleanup |
| **Phase 4** | ✅ Complete | Security hardening (OpenWrt + Proxmox), firewall, validation |
| **Phase 5** | 🔜 Planned | Ansible playbook validation, check-mode testing, CI integration |

**Last Updated**: 2026-11-12

---

## Hardware & Software Stack

### Hardware

| Component | Specification | Role |
|-----------|---------------|------|
| **Router** | TP-Link Archer C20 v5 (16MB Flash, 64MB RAM, MT7628AN + MT76x2) | OpenWrt 25.12.6 - Gateway, DHCP, DNS, WiFi AP, WWAN client |
| **Server** | Dell Inspiron N4010 (i3-M380, 4GB RAM, 238GB SSD + 465GB HDD) | Proxmox VE 9.2.2 - Hypervisor |
| **Control** | MacBook Pro (macOS) | Ansible controller, Tailscale client |

### Software

| Layer | Technology | Version |
|-------|------------|---------|
| **Orchestration** | Ansible Core | 2.21.6 |
| **Collections** | community.openwrt, community.general, ansible.utils, community.docker | Latest |
| **Router OS** | OpenWrt | 25.12.6 (r33311-981c978eff) |
| **Hypervisor** | Proxmox VE | 9.2.2 |
| **Remote Access** | Tailscale | 1.78+ |
| **Secrets** | Ansible Vault | AES256 |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           INTERNET (College Wi-Fi)                          │
│                          SSID: Rishihood_Learner                            │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │ Wi-Fi (WWAN)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TP-Link Archer C20 v5  ── OpenWrt 25.12.6                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  WWAN: phy1-sta0 (10.7.28.50/19) ──► Upstream GW: 10.7.0.1          │   │
│  │  LAN:  br-lan (192.168.1.1/24) ──► Ethernet ──► Proxmox             │   │
│  │  2.4GHz AP: radio0 (CH1, WPA2)                                      │   │
│  │  5GHz STA:  radio1 (CH149, upstream)                                │   │
│  │  5GHz AP:   radio1 (DISABLED - channel conflict with STA)           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │ Ethernet (192.168.1.0/24)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Proxmox VE 9.2.2  ── Dell Inspiron N4010                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  vmbr0: 192.168.1.10/24 (bridge: enx00e04c685ea8)                   │   │
│  │  Tailscale: 100.122.224.81 (tailscale0)                             │   │
│  │  Storage: local-lvm (151GB), local (72GB), hdd-storage (479GB)      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │ Tailscale (100.x.x.x)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MacBook Pro  ── Tailscale Client                    │
│  Tailscale IP: 100.110.79.92                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Trust Boundaries

| Zone | CIDR | Trust Level | Access |
|------|------|-------------|--------|
| **WWAN (Upstream)** | 10.7.0.0/19 | **Untrusted** | Internet only; no management |
| **LAN** | 192.168.1.0/24 | **Trusted** | Full management, SSH, Web UI |
| **Tailscale** | 100.64.0.0/10 | **Trusted** | Full management via overlay |
| **Proxmox Tailscale** | 100.122.224.81/32 | **Trusted** | SSH, Web UI (8006), SPICE (3128) |
| **OpenWrt LAN** | 192.168.1.1/24 | **Trusted** | SSH, LuCI (HTTPS only) |

---

## Network Topology

### Physical Connections

```
College Wi-Fi (Rishihood_Learner)
        │
        │ 5GHz Wi-Fi (CH149, WPA2)
        ▼
┌────────────────────────┐
│  TP-Link Archer C20 v5 │
│  (OpenWrt 25.12.6)     │
│                        │
│  phy1-sta0: 10.7.28.50 │◄── WWAN (upstream client)
│  br-lan: 192.168.1.1   │
│  eth0.1 (VLAN1) ──► LAN ports 1-4
│                        │
│  Port 1 ──► Ethernet ──► Proxmox (enx00e04c685ea8)
└────────────────────────┘
```

### IP Addressing

| Device | Hostname | Interface | IP Address | Method |
|--------|----------|-----------|------------|--------|
| OpenWrt | main-router | br-lan | 192.168.1.1/24 | Static |
| OpenWrt | main-router | phy1-sta0 | 10.7.28.50/19 | DHCP (WWAN) |
| Proxmox | pve-host | vmbr0 | 192.168.1.10/24 | Static |
| Proxmox | pve-host | tailscale0 | 100.122.224.81/32 | Tailscale |
| MacBook | macbook | en0 | 100.110.79.92/32 | Tailscale |
| DHCP Pool | - | br-lan | 192.168.1.100-250 | Dynamic |

### Reserved Static DHCP (Configured in Ansible, Not Yet Applied to Router)

| IP | Device | MAC | Purpose |
|----|--------|-----|---------|
| 192.168.1.10 | pve-host | 00:e0:4c:68:5e:a8 | Proxmox management |
| 192.168.1.50 | macbook | 34:66:91:ef:4e:2d | Control machine |

---

## Security Approach

### Defense in Depth

1. **Network Segmentation**: Management services bound to LAN interface only
2. **Firewall Default-Deny**: Proxmox `policy_in: DROP`, OpenWrt WAN zone `input: REJECT`
3. **Key-Only SSH**: Ed25519 for Proxmox, RSA for OpenWrt (dropbear limitation)
4. **Tailscale Overlay**: All remote access via authenticated WireGuard mesh
5. **HTTPS Enforcement**: LuCI redirects HTTP → HTTPS
5. **Regulatory Compliance**: India (IN) domain set on both radios
6. **rpcbind Disabled**: Unnecessary RPC service removed from Proxmox

### Hardening Summary

| Component | Change | Status |
|-----------|--------|--------|
| OpenWrt LuCI | Bind to 192.168.1.1 only | ✅ |
| OpenWrt Dropbear | Bind to LAN interface only | ✅ |
| OpenWrt LuCI | HTTPS redirect enabled | ✅ |
| OpenWrt SSH | Key-only auth (password disabled) | ✅ |
| OpenWrt Regulatory | India (IN) on both radios | ✅ |
| OpenWrt 5GHz AP | Disabled (channel conflict) | ✅ |
| OpenWrt Timezone | Asia/Kolkata | ✅ |
| Proxmox rpcbind | Stopped, disabled, masked | ✅ |
| Proxmox Firewall | Enabled with minimal ruleset | ✅ |
| Proxmox Management | Firewall-restricted to LAN + Tailscale | ✅ |

---

## Automation Approach

### Ansible Design Principles

- **Separation of Concerns**: Global settings in `group_vars/all.yml`, hardware-specific in `group_vars/openwrt.yml` and `proxmox.yml`
- **Idempotency**: All tasks use native modules (uci, template, copy) with proper change detection
- **Hardware Abstraction**: Radio names, interface names, MAC addresses in variables for migration portability
- **Secrets Management**: Ansible Vault for WiFi passwords, SSH keys, Proxmox credentials
- **Check Mode**: All playbooks support `--check --diff` for safe dry runs

### Repository Structure

```
Self.Hosted-Homelab/
├── ansible.cfg                 # Ansible configuration (per-host SSH)
├── requirements.yml            # Collection dependencies
├── inventory/
│   ├── hosts.yml              # Inventory with ProxyJump config
│   └── group_vars/
│       ├── all.yml            # Global vars (network, DNS, DHCP, firewall, SSH)
│       ├── openwrt.yml        # Hardware-specific (router, wifi, LAN, WAN, firewall_zones)
│       └── proxmox.yml        # Hardware-specific (bridges, storage, VM defaults)
├── playbooks/
│   ├── verify.yml             # Read-only verification
│   ├── backup.yml             # Backup automation
│   ├── openwrt.yml            # Main OpenWrt playbook
│   └── proxmox.yml            # Main Proxmox playbook
├── roles/
│   ├── openwrt/               # Modular tasks (system, network, wireless, dhcp, dns, firewall, ssh, services, commit)
│   └── proxmox/               # Modular tasks (network, storage, node, vm_defaults, backup)
├── configs/                    # Configuration templates
├── docs/                       # Documentation (this structure)
├── secrets/                    # Encrypted vault + password (gitignored)
└── backups/                    # Local backups (gitignored)
```

---

## Backup & Recovery Approach

### Backup Strategy

| Target | Method | Location | Retention |
|--------|--------|----------|-----------|
| **OpenWrt** | `sysupgrade -b` + UCI configs | `backups/openwrt/YYYY-MM-DD/` | 7 daily / 4 weekly / 12 monthly |
| **Proxmox** | `/etc/network/interfaces`, `/etc/pve/`, `/etc/pve/storage.cfg` | `backups/proxmox/YYYY-MM-DD/` | Same |

### Recovery Procedures

| Scenario | Procedure |
|----------|-----------|
| **Router failure** | Flash OpenWrt → `sysupgrade -r backup.tar.gz` → reboot |
| **Proxmox failure** | Reinstall Proxmox → restore `/etc/pve`, `/etc/network/interfaces` → reboot |
| **Control machine loss** | Clone repo → restore `secrets/` from password manager → run playbooks |

### Verification

- Quarterly restore tests in isolated environment
- Backup integrity checked via `tar -tzf`
- All backups timestamped and versioned

---

## Completed Phases

### Phase 3: Backup & Safe Security Improvements ✅
- OpenWrt sysupgrade backup (7.5 KB, 32 files) + UCI configs (73 KB)
- Proxmox network, storage, cluster config backups
- Overlay storage cleanup: removed duplicate ROM files (~20 KB freed)
- Ansible Vault created with WiFi passwords, SSH public key, Proxmox password
- SSH keys deployed (ed25519 to Proxmox, RSA to OpenWrt)
- Firewall verified: LuCI/SSH/DNS blocked from WWAN by OpenWrt firewall

### Phase 4: Security Hardening & Validation ✅
- **OpenWrt**: LuCI/Dropbear bound to LAN, HTTPS redirect, key-only SSH, IN regulatory domain, 5GHz AP disabled, Asia/Kolkata timezone
- **Proxmox**: rpcbind removed, firewall enabled with minimal ruleset (LAN + Tailscale for SSH 22, Web UI 8006)
- All connectivity validated: Tailscale, SSH, Web UI, ProxyJump, DNS, NTP
- External blocking verified via firewall simulation

---

## Current Limitations

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| **OpenWrt overlay 93% full** (72 KB free) | Cannot install packages, limited config changes | No opkg; avoid changes requiring overlay space |
| **No opkg/package manager** | Cannot install python3, tools, security updates | Use raw Ansible modules; minimal base image |
| **5GHz AP disabled** | No 5GHz local WiFi | Hardware limitation: single radio cannot do STA+AP on different channels |
| **DHCP reservations not applied** | Proxmox/MacBook get dynamic IPs | Apply via Ansible playbook when overlay space permits |
| **Proxmox Web UI on *:8006** | Listens on all interfaces (firewall restricts) | LISTEN_IP not supported; firewall is control plane |
| **No VMs/containers yet** | Storage unused | Ready for workloads |
| **Self-signed TLS certificates** | Browser warnings on LuCI/Proxmox | Acceptable for private lab; could use ACME later |

---

## Roadmap for Phase 5

1. **Ansible Validation**: Run `verify.yml`, `backup.yml`, `openwrt.yml` in `--check` mode with vault
2. **Fix Ansible Issues**: 
   - Update `ansible.cfg` callback plugin (remove deprecated `callback_whitelist`)
   - Create missing `interfaces.j2` template for Proxmox
   - Fix `group_vars/all.yml` firewall defaults (`input: REJECT` for WAN)
   - Update `group_vars/openwrt.yml` firewall zones to include `wan6`, `wwan`
3. **Apply DHCP Reservations**: Push static leases for Proxmox and MacBook
4. **CI Integration**: GitHub Actions for syntax check, lint, molecule tests
5. **Monitoring**: Prometheus node exporter, basic alerting
6. **Documentation**: Complete operational runbooks

---

## Documentation Navigation

| Document | Purpose |
|----------|---------|
| [Architecture Overview](docs/architecture/overview.md) | High-level system design |
| [Network Topology](docs/architecture/network-topology.md) | Physical/logical network diagrams |
| [Hardware Inventory](docs/architecture/hardware-inventory.md) | Device specifications |
| [IP Addressing](docs/architecture/ip-addressing.md) | Complete IP plan |
| [Security Architecture](docs/architecture/security-architecture.md) | Threat model, controls |
| [Prerequisites](docs/setup/prerequisites.md) | Requirements to reproduce |
| [Proxmox Setup](docs/setup/proxmox-setup.md) | Installation & config |
| [OpenWrt Setup](docs/setup/openwrt-setup.md) | Installation & config |
| [Remote Access](docs/setup/remote-access-setup.md) | Tailscale + ProxyJump |
| [Ansible Setup](docs/setup/ansible-setup.md) | Controller configuration |
| [Daily Operations](docs/operations/daily-operations.md) | Routine tasks |
| [Network Management](docs/operations/network-management.md) | Interface, routing, DHCP |
| [Firewall Management](docs/operations/firewall-management.md) | Rules, testing, rollback |
| [Backup & Restore](docs/operations/backup-and-restore.md) | Procedures & verification |
| [Troubleshooting](docs/operations/troubleshooting.md) | Common issues & fixes |
| [Phase 3 Report](docs/phases/phase-3.md) | Backup & SSH keys |
| [Phase 4 Report](docs/phases/phase-4.md) | Security hardening |
| [Architecture Decisions](docs/decisions/architecture-decisions.md) | ADRs |
| [Security Decisions](docs/decisions/security-decisions.md) | Hardening rationale |

---

## Reproducing This Project

### Prerequisites
- macOS/Linux control machine with Python 3.11+
- Ansible 2.18+ in virtual environment
- Tailscale account and auth key
- OpenWrt-supported router (Archer C20 v5 or similar)
- Proxmox VE compatible hardware

### Quick Start

```bash
# 1. Clone repository
git clone https://github.com/youruser/Self.Hosted-Homelab.git
cd Self.Hosted-Homelab

# 2. Set up virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install dependencies
pip install ansible
ansible-galaxy collection install -r requirements.yml

# 4. Configure secrets (create vault)
echo "your-vault-password" > secrets/vault_pass.txt
chmod 600 secrets/vault_pass.txt
ansible-vault create secrets/vault.yml --vault-password-file secrets/vault_pass.txt
# Add: vault_wifi_2g_password, vault_wifi_5g_password, vault_ssh_public_key, vault_proxmox_password

# 5. Update inventory/hosts.yml with your IPs and key paths

# 6. Verify connectivity
ansible all -m ping

# 7. Run verification (read-only)
ansible-playbook playbooks/verify.yml --vault-password-file secrets/vault_pass.txt

# 8. Backup current state
ansible-playbook playbooks/backup.yml --vault-password-file secrets/vault_pass.txt

# 9. Apply configuration (after review)
ansible-playbook playbooks/openwrt.yml --vault-password-file secrets/vault_pass.txt
ansible-playbook playbooks/proxmox.yml --vault-password-file secrets/vault_pass.txt
```

---

## License

MIT License - Feel free to adapt for your own homelab.

---

*This README reflects the project state as of Phase 4 completion (2026-11-12). See [docs/phases/](docs/phases/) for detailed phase reports.*