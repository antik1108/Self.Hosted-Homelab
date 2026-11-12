# Prerequisites

## Control Machine Requirements

### Hardware
- macOS, Linux, or Windows (WSL2) machine
- Minimum 4 GB RAM, 10 GB free disk space
- Network access to Tailscale and homelab LAN

### Software

| Tool | Minimum Version | Installation |
|------|----------------|--------------|
| **Python** | 3.11+ | `brew install python` / `apt install python3` |
| **Ansible Core** | 2.18+ | `pip install ansible` |
| **Git** | 2.30+ | `brew install git` / `apt install git` |
| **Tailscale** | 1.78+ | `brew install --cask tailscale` / [tailscale.com/download](https://tailscale.com/download) |
| **SSH Client** | OpenSSH 8.0+ | Built-in |

### Python Virtual Environment (Recommended)
```bash
cd Self.Hosted-Homelab
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install ansible
ansible-galaxy collection install -r requirements.yml
```

---

## Target Infrastructure Requirements

### OpenWrt Router
- **Supported Device**: Any OpenWrt-supported router (Archer C20 v5 used in this project)
- **Minimum Flash**: 16 MB (8 MB may work but limits packages)
- **Minimum RAM**: 64 MB (128 MB recommended)
- **OpenWrt Version**: 23.05+ (25.12.6 used here)
- **Network**: Must support VLANs for WAN/LAN separation
- **Wireless**: 2.4GHz radio required; 5GHz optional

### Proxmox Host
- **CPU**: x86_64 with VT-x/AMD-V (Intel i3-M380 or better)
- **RAM**: 4 GB minimum (8 GB+ recommended for VMs)
- **Storage**: 64 GB+ SSD for OS + VM storage
- **Network**: At least 1 Ethernet port (GbE recommended)
- **Proxmox VE**: 8.0+ (9.2.2 used here)

### Network Environment
- **LAN Subnet**: 192.168.1.0/24 (configurable in `group_vars/all.yml`)
- **Upstream Internet**: Ethernet WAN or Wi-Fi client (WWAN)
- **Tailscale**: Account with at least 2 devices (router not required on Tailscale)

---

## Repository Setup

### 1. Clone Repository
```bash
git clone https://github.com/youruser/Self.Hosted-Homelab.git
cd Self.Hosted-Homelab
```

### 2. Set Up Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install ansible
ansible-galaxy collection install -r requirements.yml
```

### 3. Configure Secrets (Ansible Vault)
```bash
# Create vault password file (gitignored)
echo "your-strong-vault-password" > secrets/vault_pass.txt
chmod 600 secrets/vault_pass.txt

# Create encrypted vault
ansible-vault create secrets/vault.yml --vault-password-file secrets/vault_pass.txt
```

**Required Vault Variables:**
```yaml
vault_wifi_2g_password: "your-2g-wifi-password"
vault_wifi_5g_password: "your-5g-wifi-password"
vault_ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your@email.com"
vault_proxmox_password: "your-proxmox-root-password"
```

### 4. Update Inventory
Edit `inventory/hosts.yml` with your actual IPs and key paths:
```yaml
openwrt:
  hosts:
    main-router:
      ansible_host: 192.168.1.1          # Your OpenWrt LAN IP
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      ansible_ssh_common_args: '-o ProxyCommand="ssh -W %h:%p -i ~/.ssh/id_ed25519 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@YOUR_PROXMOX_TAILSCALE_IP"'

proxmox:
  hosts:
    pve-host:
      ansible_host: YOUR_PROXMOX_TAILSCALE_IP  # e.g., 100.122.224.81
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

### 5. Update Hardware-Specific Variables
Edit `inventory/group_vars/openwrt.yml`:
```yaml
router:
  model: "Your Router Model"
  management_interface: "br-lan"
  wan_interface: "wan"
  lan_interface: "lan"

wifi:
  radio0:
    ssid: "Your-2G-SSID"
  radio1:
    ssid: "Your-5G-SSID"
    disabled: true  # If same radio used for WWAN
```

Edit `inventory/group_vars/proxmox.yml`:
```yaml
proxmox:
  management_ip: "192.168.1.10"
  management_interface: "vmbr0"

bridges:
  vmbr0:
    bridge_ports: "your-physical-interface"  # e.g., eno1, enx00e04c685ea8
```

---

## Tailscale Setup

### 1. Create Tailscale Account
- Sign up at [tailscale.com](https://tailscale.com)
- Create a tailnet (personal or organization)

### 2. Install Tailscale on Proxmox
```bash
# On Proxmox host
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --authkey=tskey-auth-xxxx  # Or use OAuth
```

### 3. Install Tailscale on Control Machine
```bash
# macOS
brew install --cask tailscale
# Or download from tailscale.com/download

# Linux
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate
tailscale up
```

### 4. Verify Connectivity
```bash
# From control machine
tailscale status
tailscale ping <proxmox-tailscale-ip>
```

---

## SSH Key Setup

### Generate Keys (if not existing)
```bash
# Ed25519 for Proxmox (modern, preferred)
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_ed25519

# RSA for OpenWrt (dropbear compatibility)
ssh-keygen -t rsa -b 4096 -C "your@email.com" -f ~/.ssh/id_rsa
```

### Deploy to Proxmox
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<proxmox-tailscale-ip>
```

### Deploy to OpenWrt (via Proxmox ProxyJump)
```bash
ssh -J root@<proxmox-tailscale-ip> root@192.168.1.1 \
  "mkdir -p /etc/dropbear && cat >> /etc/dropbear/authorized_keys" \
  < ~/.ssh/id_rsa.pub
```

### Verify
```bash
# Direct to Proxmox
ssh -i ~/.ssh/id_ed25519 root@<proxmox-tailscale-ip> "echo OK"

# ProxyJump to OpenWrt
ssh -i ~/.ssh/id_rsa -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/id_ed25519 root@<proxmox-tailscale-ip>" root@192.168.1.1 "echo OK"
```

---

## Ansible Configuration

### Verify `ansible.cfg`
Key settings in this project:
```ini
[defaults]
inventory = ./inventory/hosts.yml
host_key_checking = False
interpreter_python = auto_silent
stdout_callback = default

[ssh_connection]
ssh_args = -o ControlMaster=no -o ControlPersist=60s -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no
pipelining = True
# ssh_executable = /tmp/ansible_ssh_wrapper.sh  # Per-host in inventory
```

### Test Connectivity
```bash
# Test Proxmox (direct via Tailscale)
ansible pve-host -m ping --vault-password-file secrets/vault_pass.txt

# Test OpenWrt (via ProxyJump, raw module only)
ansible main-router -m raw -a "ubus call system board" --vault-password-file secrets/vault_pass.txt
```

---

## First Run Checklist

- [ ] Repository cloned
- [ ] Virtual environment created and activated
- [ ] Ansible collections installed
- [ ] Vault created with all 4 required variables
- [ ] `inventory/hosts.yml` updated with your IPs and key paths
- [ ] `inventory/group_vars/openwrt.yml` updated with your hardware details
- [ ] `inventory/group_vars/proxmox.yml` updated with your bridge interface
- [ ] Tailscale installed and authenticated on Proxmox and control machine
- [ ] SSH keys generated and deployed to both targets
- [ ] Connectivity verified: `ansible all -m ping` (Proxmox) and raw module (OpenWrt)

---

## Next Steps

1. Run verification playbook: `ansible-playbook playbooks/verify.yml --vault-password-file secrets/vault_pass.txt`
2. Create initial backups: `ansible-playbook playbooks/backup.yml --vault-password-file secrets/vault_pass.txt`
3. Review playbooks in check mode: `ansible-playbook playbooks/openwrt.yml --check --diff --vault-password-file secrets/vault_pass.txt`
4. Apply configuration when ready

---

*See [Ansible Setup](ansible-setup.md) for detailed Ansible configuration, [Proxmox Setup](proxmox-setup.md) for Proxmox-specific steps, [OpenWrt Setup](openwrt-setup.md) for router-specific steps, and [Remote Access Setup](remote-access-setup.md) for Tailscale/ProxyJump details.*