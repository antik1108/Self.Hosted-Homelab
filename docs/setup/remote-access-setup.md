# Remote Access Setup (Tailscale + ProxyJump)

## Overview

This homelab uses **Tailscale** as a zero-trust overlay network for remote management. The control machine (MacBook) connects to Proxmox via Tailscale, then uses SSH ProxyJump to reach OpenWrt on the LAN.

```
┌─────────────────┐     Tailscale (WireGuard)      ┌──────────────────┐
│  MacBook Pro    │ ──────────────────────────────► │   Proxmox Host   │
│  100.110.79.92  │     UDP 41641 (DERP relay)      │  100.122.224.81  │
│  (Control)      │ ◄─────────────────────────────── │  (Jump Host)     │
└─────────────────┘                                 └────────┬─────────┘
                                                              │ SSH ProxyJump
                                                              ▼
                                                     ┌──────────────────┐
                                                     │   OpenWrt Router │
                                                     │  192.168.1.1     │
                                                     │  (Target)        │
                                                     └──────────────────┘
```

---

## Tailscale Configuration

### 1. Install Tailscale on Proxmox
```bash
# On Proxmox host
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --authkey=tskey-auth-xxxx  # Or use OAuth
```

### 2. Install Tailscale on Control Machine
```bash
# macOS
brew install --cask tailscale

# Linux
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate
tailscale up
```

### 3. Verify Connection
```bash
# From control machine
tailscale status
# Should show both devices as "active"

tailscale ping 100.122.224.81
# Should show latency via DERP or direct

# Direct SSH test
ssh root@100.122.224.81
```

### 4. Tailscale ACLs (Admin Panel)
In Tailscale admin console (https://login.tailscale.com/admin/machines):

1. **Disable "Allow LAN access"** for Proxmox if you want strict control
2. **Add tag** for homelab devices
3. **Configure ACLs** (example):
```json
{
  "groups": {
    "group:homelab": ["autogroup:shared"],
    "group:control": ["user:your@email.com"]
  },
  "acls": [
    {"action": "accept", "src": ["group:control"], "dst": ["group:homelab:*"]},
    {"action": "accept", "src": ["group:homelab"], "dst": ["group:homelab:*"]}
  ]
}
```

---

## SSH ProxyJump Configuration

### 1. SSH Config (~/.ssh/config)
```ssh
# Proxmox Host (direct via Tailscale)
Host pve
    HostName 100.122.224.81
    User root
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    ControlMaster auto
    ControlPersist 60s

# OpenWrt Router (via Proxmox ProxyJump)
Host openwrt
    HostName 192.168.1.1
    User root
    IdentityFile ~/.ssh/id_rsa
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    ProxyJump pve
    ControlMaster auto
    ControlPersist 60s
```

### 2. Test Connections
```bash
# Direct to Proxmox
ssh pve "echo 'Proxmox OK'"

# ProxyJump to OpenWrt
ssh openwrt "echo 'OpenWrt OK'"

# SCP via ProxyJump
scp openwrt:/etc/config/network ./network.backup
```

---

## Ansible Inventory Configuration

### inventory/hosts.yml
```yaml
all:
  vars:
    ansible_python_interpreter: auto_silent
    ansible_ssh_common_args: '-o ControlMaster=no -o ControlPersist=60s -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no'
    ansible_vault_password_file: secrets/vault_pass.txt

openwrt:
  hosts:
    main-router:
      ansible_host: 192.168.1.1
      ansible_user: root
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      ansible_ssh_common_args: >-
        -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/id_ed25519
        -o StrictHostKeyChecking=no
        -o UserKnownHostsFile=/dev/null
        root@100.122.224.81"
      ansible_python_interpreter: /usr/bin/python3

proxmox:
  hosts:
    pve-host:
      ansible_host: 100.122.224.81
      ansible_user: root
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

### Key Points:
- **Proxmox**: Direct Tailscale IP (`ansible_host: 100.122.224.81`)
- **OpenWrt**: LAN IP (`ansible_host: 192.168.1.1`) with `ProxyCommand` via Proxmox
- **OpenWrt uses RSA key** (dropbear compatibility), Proxmox uses Ed25519
- **OpenWrt has no Python** → Ansible uses `raw` module or `community.openwrt.uci`

---

## Usage Examples

### Direct Commands
```bash
# Proxmox
ansible pve-host -m shell -a "uptime" --vault-password-file secrets/vault_pass.txt

# OpenWrt (raw module only - no Python)
ansible main-router -m raw -a "ubus call system board" --vault-password-file secrets/vault_pass.txt
ansible main-router -m raw -a "df -h /overlay" --vault-password-file secrets/vault_pass.txt
```

### Playbooks
```bash
# Verify connectivity
ansible-playbook playbooks/verify.yml --vault-password-file secrets/vault_pass.txt

# Backup
ansible-playbook playbooks/backup.yml --vault-password-file secrets/vault_pass.txt

# Apply OpenWrt config
ansible-playbook playbooks/openwrt.yml --check --diff --vault-password-file secrets/vault_pass.txt

# Apply Proxmox config
ansible-playbook playbooks/proxmox.yml --check --diff --vault-password-file secrets/vault_pass.txt
```

### SSH Wrapper (Alternative)
For simpler OpenWrt access, use a wrapper script:

```bash
# /tmp/ansible_ssh_wrapper.sh
#!/bin/bash
exec ssh -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/id_ed25519 root@100.122.224.81" "$@"

chmod +x /tmp/ansible_ssh_wrapper.sh
```

Then in inventory:
```yaml
ansible_ssh_executable: /tmp/ansible_ssh_wrapper.sh
ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

---

## Troubleshooting

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| **Tailscale not connecting** | `tailscale status` shows "stopped" | `tailscale up`, check auth key, `journalctl -u tailscaled` |
| **DERP relay high latency** | `tailscale ping` shows relay | Check NAT type, enable UPnP, consider exit node |
| **ProxyJump fails** | `ssh openwrt` times out | Verify Proxmox SSH works, check `AllowTcpForwarding yes` on Proxmox |
| **Ansible raw fails** | "python3 not found" | Use `raw` module only, don't use `setup` or `ping` |
| **Host key warnings** | StrictHostKeyChecking errors | `UserKnownHostsFile=/dev/null StrictHostKeyChecking=no` in config |
| **ControlMaster issues** | Stale sockets | `ssh -O exit pve`, `ssh -O exit openwrt`, remove `~/.ssh/cm_*` |

---

## Security Notes

1. **Tailscale ACLs** control which devices can reach Proxmox
2. **Proxmox firewall** further restricts to LAN + Tailscale IP
3. **OpenWrt** only accessible via ProxyJump (no direct Tailscale)
4. **SSH keys** are Ed25519 (Proxmox) and RSA (OpenWrt)
5. **Vault password** protects WiFi passwords and SSH public key

---

## Adding New Devices

### New Proxmox Node
1. Install Tailscale
2. Add to inventory with new Tailscale IP
3. Use Ed25519 key

### New OpenWrt Router
1. Set up SSH key (RSA)
2. Add to inventory with LAN IP
3. Configure ProxyJump via existing Proxmox
4. Or: Set up Tailscale on router (if supported) for direct access

---

*See [Prerequisites](prerequisites.md) for requirements, [Proxmox Setup](proxmox-setup.md) and [OpenWrt Setup](openwrt-setup.md) for target config, [Ansible Setup](ansible-setup.md) for automation details.*