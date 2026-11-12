# Homelab Infrastructure as Code

This repository contains the Infrastructure as Code (IaC) configuration for my personal homelab, managed with Ansible.

## Architecture

```
MacBook (Control Machine)
    │
    ├── SSH/Ansible ──────────────► OpenWrt Router (Archer C20 v5)
    │                                └── Network, DHCP, DNS, Firewall, WiFi
    │
    └── SSH/Ansible ──────────────► Proxmox VE Host
                                      └── VMs, Containers, Storage, Network
```

## Project Structure

```
Self.Hosted-Homelab/
├── ansible.cfg              # Ansible configuration
├── requirements.yml         # Ansible collections/roles dependencies
├── inventory/
│   ├── hosts.yml           # Host inventory
│   └── group_vars/         # Group variables
│       ├── all.yml         # Global variables
│       ├── openwrt.yml     # OpenWrt-specific variables
│       └── proxmox.yml     # Proxmox-specific variables
├── playbooks/
│   ├── openwrt.yml         # OpenWrt configuration playbook
│   ├── proxmox.yml         # Proxmox configuration playbook
│   ├── backup.yml          # Backup configurations
│   └── verify.yml          # Verify configurations
├── roles/
│   ├── openwrt/            # OpenWrt configuration role
│   └── proxmox/            # Proxmox configuration role
├── configs/
│   ├── network/            # Network configuration templates
│   └── services/           # Service configurations
├── docs/
│   ├── architecture.md     # This file
│   ├── network-design.md   # Network design document
│   └── migration-guide.md  # Router migration guide
└── backups/                # Local backups (gitignored)
```

## Prerequisites

- macOS (control machine)
- Python 3.13+
- Ansible 14+ (in virtual environment)
- `community.openwrt` Ansible collection
- SSH access to OpenWrt and Proxmox

## Quick Start

```bash
# Clone repository
git clone https://github.com/antik1108/Self.Hosted-Homelab.git
cd Self.Hosted-Homelab

# Set up virtual environment
python3.13 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt  # or pip install ansible
ansible-galaxy collection install -r requirements.yml

# Verify connectivity
ansible all -m ping

# Run playbooks
ansible-playbook playbooks/verify.yml     # Verify current state
ansible-playbook playbooks/backup.yml     # Backup current configs
ansible-playbook playbooks/openwrt.yml    # Configure OpenWrt
ansible-playbook playbooks/proxmox.yml    # Configure Proxmox
```

## Configuration Design

### Separation of Concerns

- **Reusable network settings** (`inventory/group_vars/all.yml`): LAN subnet, DNS, DHCP defaults
- **Hardware-specific settings** (`inventory/group_vars/openwrt.yml`, `proxmox.yml`): Interface names, radio names, MAC addresses

This separation makes router migration straightforward - only hardware-specific variables need to change.

### Secrets Management

Sensitive values are stored in Ansible Vault:

```bash
# Create vault password file
echo "your-password" > .vault_pass

# Encrypt a variable
ansible-vault encrypt_string "wifi-password" --name "vault_wifi_2g_password"

# Run with vault
ansible-playbook playbooks/openwrt.yml --vault-password-file .vault_pass
```

Required vault variables:
- `vault_wifi_2g_password` - 2.4GHz WiFi password
- `vault_wifi_5g_password` - 5GHz WiFi password
- `vault_ssh_public_key` - SSH public key for root access
- `vault_proxmox_password` - Proxmox root password (for API)

## Phases

1. **Phase 1** - Development environment setup ✓
2. **Phase 2** - Document existing network
3. **Phase 3** - Secure remote access (SSH keys)
4. **Phase 4** - Backup existing configurations
5. **Phase 5** - Automate OpenWrt
6. **Phase 6** - Automate Proxmox
7. **Phase 7** - Git and documentation
8. **Phase 8** - Test migration to new router

## Safety Rules

- Never apply changes without `verify.yml` first
- Always run `backup.yml` before making changes
- Use `--check` mode for dry runs
- Keep backups outside the repository
- Never commit secrets to Git

## Documentation

- [Network Design](docs/network-design.md)
- [Migration Guide](docs/migration-guide.md)

## License

MIT License - Feel free to adapt for your own homelab.