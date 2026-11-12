# Ansible Setup & Usage

## Overview

This project uses Ansible to manage both OpenWrt and Proxmox infrastructure as code. Key constraints:
- **OpenWrt**: No Python, no opkg → must use `raw` module or `community.openwrt.uci`
- **Proxmox**: Full Python → standard modules + `community.general.proxmox_*`
- **Connection**: Proxmox direct via Tailscale; OpenWrt via ProxyJump

---

## Repository Structure

```
Self.Hosted-Homelab/
├── ansible.cfg                 # Ansible configuration
├── inventory/
│   ├── hosts.yml              # Host inventory with ProxyJump
│   └── group_vars/
│       ├── all.yml            # Global variables
│       ├── openwrt.yml        # OpenWrt-specific variables
│       └── proxmox.yml        # Proxmox-specific variables
├── playbooks/
│   ├── verify.yml             # Connectivity verification
│   ├── backup.yml             # Backup both hosts
│   ├── openwrt.yml            # OpenWrt configuration
│   └── proxmox.yml            # Proxmox configuration
├── roles/
│   ├── openwrt/               # OpenWrt role
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── network.yml
│   │   │   ├── wireless.yml
│   │   │   ├── dhcp_dns.yml
│   │   │   ├── firewall.yml
│   │   │   ├── ssh.yml
│   │   │   ├── luci.yml
│   │   │   └── system.yml
│   │   ├── templates/
│   │   └── vars/
│   └── proxmox/               # Proxmox role
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── network.yml
│       │   ├── firewall.yml
│       │   ├── storage.yml
│       │   ├── packages.yml
│       │   └── tailscale.yml
│       └── templates/
└── secrets/
    ├── vault.yml              # Encrypted secrets (AES256)
    └── vault_pass.txt         # Vault password (600, gitignored)
```

---

## Ansible Configuration (ansible.cfg)

```ini
[defaults]
inventory = ./inventory/hosts.yml
host_key_checking = False
interpreter_python = auto_silent
stdout_callback = default
vault_password_file = ./secrets/vault_pass.txt

[ssh_connection]
ssh_args = -o ControlMaster=no -o ControlPersist=60s -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no
pipelining = True
# ssh_executable = /tmp/ansible_ssh_wrapper.sh  # Per-host in inventory instead

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

### Key Settings Explained

| Setting | Purpose |
|---------|---------|
| `host_key_checking = False` | Avoid host key prompts for ephemeral IPs |
| `interpreter_python = auto_silent` | Auto-detect Python, suppress warnings |
| `ControlMaster=no` | Disable multiplexing (causes issues with ProxyJump) |
| `ControlPersist=60s` | Keep connections alive for 60s |
| `pipelining = True` | Reduce SSH round-trips (requires sudo on target) |
| `vault_password_file` | Default vault password location |

---

## Inventory Structure

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
      ansible_ssh_private_key_file: "~/.ssh/id_rsa"
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
      ansible_ssh_private_key_file: "~/.ssh/id_ed25519"
```

### inventory/group_vars/all.yml
```yaml
# Global settings
timezone: "Asia/Kolkata"
ntp_servers:
  - "0.openwrt.pool.ntp.org"
  - "1.openwrt.pool.ntp.org"
  - "2.openwrt.pool.ntp.org"
  - "3.openwrt.pool.ntp.org"

# Network
lan_cidr: "192.168.1.0/24"
lan_gateway: "192.168.1.1"
dns_servers:
  - "1.1.1.1"
  - "8.8.8.8"
  - "9.9.9.9"
```

### inventory/group_vars/openwrt.yml
```yaml
router:
  model: "TP-Link Archer C20 v5"
  arch: "ramips/mt76x8"
  management_interface: "br-lan"
  wan_interface: "wan"
  lan_interface: "lan"

wifi:
  radio0:
    name: "radio0"
    phy: "phy0"
    band: "2g"
    channel: "1"
    htmode: "HT20"
    ssid: "HomeLab-2G"
    encryption: "psk2+ccmp"
    country: "IN"
    disabled: false
  radio1:
    name: "radio1"
    phy: "phy1"
    band: "5g"
    channel: "149"
    htmode: "VHT40"
    ssid: "Rishihood_Learner"  # WWAN STA
    encryption: "psk2"
    country: "IN"
    disabled: false
  ap_5g:
    name: "default_radio1"
    ssid: "HomeLab-5G"
    encryption: "sae-mixed"
    disabled: true  # Channel conflict with STA

dhcp:
  lan:
    start: 100
    limit: 150
    leasetime: "12h"
  static_hosts:
    - name: "proxmox-host"
      mac: "00:e0:4c:68:5e:a8"
      ip: "192.168.1.10"
    - name: "macbook"
      mac: "34:66:91:ef:4e:2d"
      ip: "192.168.1.50"

firewall:
  defaults:
    syn_flood: true
    input: "REJECT"
    output: "ACCEPT"
    forward: "REJECT"
  zones:
    - name: "lan"
      input: "ACCEPT"
      output: "ACCEPT"
      forward: "ACCEPT"
      network: "lan"
    - name: "wan"
      input: "REJECT"
      output: "ACCEPT"
      forward: "DROP"
      masq: true
      mtu_fix: true
      network: ["wan", "wan6", "wwan"]
  forwardings:
    - src: "lan"
      dest: "wan"
```

### inventory/group_vars/proxmox.yml
```yaml
proxmox:
  hostname: "pve-host"
  management_ip: "192.168.1.10/24"
  management_gateway: "192.168.1.1"
  management_interface: "vmbr0"

bridges:
  vmbr0:
    name: "vmbr0"
    type: "static"
    ip: "192.168.1.10/24"
    gateway: "192.168.1.1"
    bridge_ports: "enx00e04c685ea8"
    bridge_stp: "off"
    bridge_fd: "0"

firewall:
  enable: true
  policy_in: "DROP"
  policy_out: "ACCEPT"
  policy_forward: "ACCEPT"
  management_network: "192.168.1.0/24"
  allowed_ports:
    - "22"      # SSH
    - "8006"    # Web UI
    - "3128"    # SPICE proxy
  tailscale_ip: "100.122.224.81/32"

storage:
  local:
    type: "dir"
    path: "/var/lib/vz"
    content: "vztmpl,iso,backup"
  local-lvm:
    type: "lvmthin"
    vgname: "pve"
    content: "rootdir,images"
  hdd-storage:
    type: "dir"
    path: "/mnt/pve/hdd-storage"
    content: "iso,backup"

packages:
  - "tailscale"
  - "vim"
  - "htop"
  - "iotop"

vm_defaults:
  memory: 2048
  cores: 2
  cpu: "host"
  net0: "virtio,bridge=vmbr0"
  scsihw: "virtio-scsi-pci"
  scsi0: "local-lvm:32"
  ostype: "l26"
  onboot: true
  agent: "1,fstrim_cloned_disks=1"
```

---

## Vault (Secrets Management)

### Create Vault
```bash
# Create password file
echo "your-strong-password" > secrets/vault_pass.txt
chmod 600 secrets/vault_pass.txt

# Create vault
ansible-vault create secrets/vault.yml --vault-password-file secrets/vault_pass.txt
```

### Required Vault Variables
```yaml
vault_wifi_2g_password: "lundlemera"
vault_wifi_5g_password: "HomeLab5G-SecurePass2026!"
vault_ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your@email.com"
vault_proxmox_password: "tamannahazari"
```

### View/Edit Vault
```bash
# View
ansible-vault view secrets/vault.yml --vault-password-file secrets/vault_pass.txt

# Edit
ansible-vault edit secrets/vault.yml --vault-password-file secrets/vault_pass.txt

# Rekey (change password)
ansible-vault rekey secrets/vault.yml --vault-password-file secrets/vault_pass.txt --new-vault-password-file new_pass.txt
```

---

## Playbooks

### playbooks/verify.yml
```yaml
---
- name: Verify Proxmox connectivity
  hosts: pve-host
  gather_facts: true
  tasks:
    - name: Ping
      ansible.builtin.ping:
    - name: System info
      ansible.builtin.setup:
      register: pve_facts

- name: Verify OpenWrt connectivity (raw only)
  hosts: main-router
  gather_facts: false
  tasks:
    - name: Check system board
      ansible.builtin.raw: "ubus call system board"
      register: sys_board
    - name: Show result
      ansible.builtin.debug:
        var: sys_board.stdout
```

### playbooks/backup.yml
```yaml
---
- name: Backup Proxmox
  hosts: pve-host
  become: true
  tasks:
    - name: Backup network config
      ansible.builtin.copy:
        content: "{{ lookup('file', '/etc/network/interfaces') }}"
        dest: "{{ backup_dir }}/proxmox/interfaces-{{ ansible_date_time.iso8601 }}.bak"
      delegate_to: localhost
      vars:
        backup_dir: "{{ playbook_dir }}/../backups/proxmox/{{ ansible_date_time.date }}"
    - name: Backup firewall config
      ansible.builtin.copy:
        content: "{{ lookup('file', '/etc/pve/nodes/homelab/host.fw') }}"
        dest: "{{ backup_dir }}/proxmox/host.fw-{{ ansible_date_time.iso8601 }}.bak"
      delegate_to: localhost

- name: Backup OpenWrt
  hosts: main-router
  gather_facts: false
  tasks:
    - name: Create sysupgrade backup
      ansible.builtin.raw: "sysupgrade -b /tmp/openwrt-backup-{{ ansible_date_time.iso8601 }}.tar.gz"
    - name: Fetch backup
      ansible.builtin.fetch:
        src: "/tmp/openwrt-backup-{{ ansible_date_time.iso8601 }}.tar.gz"
        dest: "{{ backup_dir }}/openwrt/"
        flat: true
      vars:
        backup_dir: "{{ playbook_dir }}/../backups/openwrt/{{ ansible_date_time.date }}"
    - name: Dump UCI config
      ansible.builtin.raw: "uci show"
      register: uci_config
    - name: Save UCI config
      ansible.builtin.copy:
        content: "{{ uci_config.stdout }}"
        dest: "{{ backup_dir }}/openwrt/uci-configs-{{ ansible_date_time.iso8601 }}.txt"
      delegate_to: localhost
```

### playbooks/openwrt.yml
```yaml
---
- name: Configure OpenWrt Router
  hosts: main-router
  gather_facts: false
  vars_files:
    - ../secrets/vault.yml
  roles:
    - openwrt
```

### playbooks/proxmox.yml
```yaml
---
- name: Configure Proxmox Host
  hosts: pve-host
  become: true
  vars_files:
    - ../secrets/vault.yml
  roles:
    - proxmox
```

---

## Roles

### OpenWrt Role (roles/openwrt/)

Uses `community.openwrt.uci` module for configuration:

```yaml
# roles/openwrt/tasks/network.yml
- name: Configure LAN interface
  community.openwrt.uci:
    config: network
    section: lan
    type: interface
    values:
      device: br-lan
      proto: static
      ipaddr: "{{ lan_gateway }}"
      netmask: 255.255.255.0
      ip6assign: "60"

- name: Configure WAN interface
  community.openwrt.uci:
    config: network
    section: wan
    type: interface
    values:
      device: eth0.2
      proto: dhcp

- name: Configure WWAN interface
  community.openwrt.uci:
    config: network
    section: wwan
    type: interface
    values:
      proto: dhcp
      device: phy1-sta0
```

### Proxmox Role (roles/proxmox/)

Uses standard modules + templates:

```yaml
# roles/proxmox/tasks/network.yml
- name: Write network interfaces
  ansible.builtin.template:
    src: interfaces.j2
    dest: /etc/network/interfaces
    mode: "0644"
    backup: true
  notify: reload networking

- name: Reload networking
  ansible.builtin.systemd:
    name: networking
    state: restarted
```

---

## Running Playbooks

### Check Mode (Dry Run)
```bash
# OpenWrt
ansible-playbook playbooks/openwrt.yml --check --diff --vault-password-file secrets/vault_pass.txt

# Proxmox
ansible-playbook playbooks/proxmox.yml --check --diff --vault-password-file secrets/vault_pass.txt
```

### Apply Changes
```bash
# OpenWrt
ansible-playbook playbooks/openwrt.yml --vault-password-file secrets/vault_pass.txt

# Proxmox
ansible-playbook playbooks/proxmox.yml --vault-password-file secrets/vault_pass.txt
```

### Limit to Specific Host/Tag
```bash
# Single host
ansible-playbook playbooks/openwrt.yml --limit main-router --vault-password-file secrets/vault_pass.txt

# Specific tag
ansible-playbook playbooks/openwrt.yml --tags firewall --vault-password-file secrets/vault_pass.txt
```

### Verbose Output
```bash
ansible-playbook playbooks/openwrt.yml -vvv --vault-password-file secrets/vault_pass.txt
```

---

## Common Tasks

### OpenWrt UCI via Ansible
```yaml
- name: Set wireless country
  community.openwrt.uci:
    config: wireless
    section: radio0
    type: wifi-device
    values:
      country: IN
```

### Proxmox Firewall
```yaml
- name: Configure host firewall
  ansible.builtin.copy:
    dest: /etc/pve/nodes/homelab/host.fw
    content: |
      [RULES]
      IN SSH(ACCEPT) -source {{ firewall.management_network }}
      IN ACCEPT -p tcp -dport 8006 -source {{ firewall.management_network }}
      IN ACCEPT -p tcp -dport 8006 -source {{ firewall.tailscale_ip }}
      OUT ACCEPT
    mode: "0644"
  notify: restart pve-firewall
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **"python3 not found" on OpenWrt** | Use `raw` module or `community.openwrt.uci`; don't use `setup` or `ping` |
| **ProxyJump fails** | Check `ansible_ssh_common_args` in inventory, verify Proxmox SSH works |
| **Vault decrypt fails** | Verify `vault_pass.txt` exists and is 600, try `--ask-vault-pass` |
| **Template errors** | Check Jinja2 syntax, test with `ansible-playbook --syntax-check` |
| **Idempotency failures** | Use `community.openwrt.uci` for OpenWrt (handles idempotency) |
| **Connection timeout** | Increase `ControlPersist`, check Tailscale status |

---

## Best Practices

1. **Always run `--check --diff` first**
2. **Use tags** for targeted runs: `--tags firewall,network`
3. **Commit vault changes** only when password hasn't changed
4. **Backup before major changes**: `ansible-playbook playbooks/backup.yml`
5. **Test on single host** with `--limit` before full rollout
6. **Use `raw` for OpenWrt commands** that don't have modules
7. **Keep `host_key_checking = False`** for lab environments

---

## Adding New Modules/Collections

```yaml
# requirements.yml
collections:
  - community.openwrt
  - community.general
  - community.libvirt
```

```bash
ansible-galaxy collection install -r requirements.yml
```

---

*See [Prerequisites](prerequisites.md) for setup, [Remote Access Setup](remote-access-setup.md) for connection details, [Proxmox Setup](proxmox-setup.md) and [OpenWrt Setup](openwrt-setup.md) for target configuration.*