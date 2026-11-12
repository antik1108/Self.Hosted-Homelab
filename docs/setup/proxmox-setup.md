# Proxmox VE Setup

## Installation

### 1. Download Proxmox VE ISO
- Download from [proxmox.com/downloads](https://www.proxmox.com/downloads)
- Version used in this project: **Proxmox VE 9.2.2** (ISO: `proxmox-ve_9.2-2.iso`)

### 2. Create Boot Media
```bash
# Linux/macOS
sudo dd if=proxmox-ve_9.2-2.iso of=/dev/sdX bs=4M status=progress && sync

# Windows: Use Rufus or BalenaEtcher
```

### 3. Install on Target Hardware
1. Boot from USB
2. Select "Install Proxmox VE"
3. Accept EULA
3. Select target disk (SSD recommended)
4. Configure:
   - **Country/Timezone**: Your location
   - **Root Password**: Strong password (store in vault)
   - **Email**: For notifications
   - **Hostname**: `pve-host` (or your choice)
   - **Management IP**: Static `192.168.1.10/24`, Gateway `192.168.1.1`
   - **DNS**: `192.168.1.1` (OpenWrt) or `1.1.1.1`
5. Install and reboot

### 4. Post-Install Configuration

#### Access Web UI
```
https://192.168.1.10:8006
User: root
Password: <your-root-password>
```

#### Disable Enterprise Repository (No Subscription)
```bash
# Edit sources
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Update
apt update && apt full-upgrade -y
```

#### Install Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --authkey=tskey-auth-xxxx  # Or use OAuth
```

#### Configure Tailscale for Persistent Connection
```bash
# Enable Tailscale SSH (optional)
tailscale up --ssh

# Verify
tailscale status
```

---

## Network Configuration

### Current Setup (vmbr0)
The Proxmox host uses a single bridge for management and VM traffic:

```bash
# /etc/network/interfaces (managed by Ansible)
auto lo
iface lo inet loopback

iface enx00e04c685ea8 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports enx00e04c685ea8
    bridge-stp off
    bridge-fd 0
```

### Ansible-Managed Template
The project uses a Jinja2 template (`roles/proxmox/templates/interfaces.j2`) which should be created:

```jinja
# {{ ansible_managed }}

auto lo
iface lo inet loopback

{% for iface, config in bridges.items() %}
auto {{ iface }}
iface {{ iface }} inet {{ config.type }}
    {% if config.type == 'static' %}
    address {{ config.ip }}
    gateway {{ config.gateway }}
    {% endif %}
    bridge-ports {{ config.bridge_ports }}
    bridge-stp {{ config.bridge_stp }}
    bridge-fd {{ config.bridge_fd }}
    {% if config.comment is defined %}# {{ config.comment }}{% endif %}
{% endfor %}
```

### Variables (inventory/group_vars/proxmox.yml)
```yaml
bridges:
  vmbr0:
    name: "vmbr0"
    type: "static"
    ip: "192.168.1.10/24"
    gateway: "192.168.1.1"
    bridge_ports: "enx00e04c685ea8"  # Physical NIC name
    bridge_stp: "off"
    bridge_fd: "0"
    comment: "Management bridge - LAN"
```

---

## Storage Configuration

### Current Layout
| Storage | Type | Path/Device | Purpose |
|---------|------|-------------|---------|
| local | dir | `/var/lib/vz` | Templates, ISOs, backups |
| local-lvm | lvmthin | `pve/data` (LVM-thin) | VM disks, containers |
| hdd-storage | dir | `/mnt/pve/hdd-storage` | Bulk storage, ISOs |

### LVM-Thin Setup (Already Configured)
```bash
# Verify
pvs
vgs
lvs

# Expected: pve VG with root, swap, data (thin pool)
```

### Adding Storage via Ansible
```yaml
# inventory/group_vars/proxmox.yml
storage:
  local-lvm:
    type: "lvmthin"
    vgname: "pve"
    content: "rootdir,images"
  local:
    type: "dir"
    path: "/var/lib/vz"
    content: "vztmpl,iso,backup"
```

---

## Firewall Setup (pve-firewall)

### Enable Cluster Firewall
```bash
# Create cluster.fw
cat > /etc/pve/firewall/cluster.fw << 'EOF'
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
policy_forward: ACCEPT

[ALIASES]
local_network 192.168.1.0/24
EOF
```

### Configure Host Firewall
```bash
# Create host.fw
cat > /etc/pve/nodes/homelab/host.fw << 'EOF'
[RULES]
IN SSH(ACCEPT) -source 192.168.1.0/24
IN ACCEPT -p tcp -dport 8006 -source 192.168.1.0/24
OUT ACCEPT
EOF
```

### Apply and Verify
```bash
pve-firewall compile
pve-firewall start
pve-firewall status
```

### Test Rules
```bash
# From allowed LAN IP
curl -k https://192.168.1.10:8006
ssh root@192.168.1.10

# From Tailscale
curl -k https://100.122.224.81:8006
ssh root@100.122.224.81
```

---

## Tailscale Integration

### Configuration
```bash
# Install
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate (use auth key or OAuth)
tailscale up --authkey=tskey-auth-xxxx

# Optional: Enable SSH access via Tailscale
tailscale up --ssh
```

### Verify
```bash
tailscale status
# Should show: 100.122.224.81  homelab  active

# Test from control machine
ssh root@100.122.224.81
curl -k https://100.122.224.81:8006
```

### Tailscale Firewall Interaction
Tailscale creates its own `ts-input` iptables chain that runs **before** pve-firewall:
- All traffic from `tailscale0` interface: ACCEPT
- UDP 41641 (WireGuard/DERP): ACCEPT from anywhere
- Tailscale subnet (100.64.0.0/10) on non-tailscale0: DROP

This means Proxmox firewall rules for Tailscale IPs are **supplementary**; the primary access control is Tailscale's own ACLs.

---

## VM/CT Preparation

### Default VM Settings (Ansible Variables)
```yaml
# inventory/group_vars/proxmox.yml
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

### Cloud-Init Template (Recommended)
1. Download cloud image (Ubuntu, Debian, etc.)
2. Create VM with cloud-init drive
3. Convert to template
4. Clone for new VMs

---

## Backup Configuration

### Proxmox Backup Server (PBS) - Optional
If you have a separate backup server:
```bash
# Add PBS storage
# Datacenter → Storage → Add → Proxmox Backup Server
```

### Local Backup Schedule (vzdump)
```yaml
# inventory/group_vars/proxmox.yml
backup:
  storage: "local"
  mode: "snapshot"
  compression: "zstd"
  schedule: "0 2 * * *"  # Daily 2 AM
  retention: 7
```

### Manual Backup
```bash
# Backup all VMs/CTs
vzdump --all --storage local --mode snapshot --compress zstd

# Backup specific VM
vzdump 100 --storage local --mode snapshot --compress zstd
```

---

## Verification Checklist

- [ ] Proxmox VE installed and accessible at `https://192.168.1.10:8006`
- [ ] Enterprise repo disabled, no-subscription repo enabled
- [ ] System updated (`apt full-upgrade`)
- [ ] Tailscale installed and connected
- [ ] vmbr0 configured with static IP `192.168.1.10/24`
- [ ] Storage configured (local, local-lvm, hdd-storage)
- [ ] pve-firewall enabled with cluster.fw and host.fw
- [ ] Firewall rules allow SSH (22) and Web UI (8006) from LAN + Tailscale
- [ ] Tailscale accessible from control machine
- [ ] SSH key authentication working
- [ ] Backup schedule configured (or manual process documented)

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Web UI not accessible** | Check `pveproxy` service, firewall rules, `systemctl status pveproxy` |
| **Tailscale not connecting** | Check `tailscale status`, `journalctl -u tailscaled`, verify auth key |
| **Firewall blocking access** | `pve-firewall stop` to test, then check rules with `pve-firewall compile` |
| **VM network not working** | Verify vmbr0 bridge ports, check VM network config (virtio, bridge=vmbr0) |
| **Storage full** | `pvesm status`, clean old backups, ISOs, templates |
| **Cluster config errors** | `pvecm status`, check `/etc/pve/corosync.conf` |

---

*See [Architecture Overview](../architecture/overview.md) for context, [Network Topology](../architecture/network-topology.md) for diagrams, [Prerequisites](prerequisites.md) for requirements, and [Ansible Setup](ansible-setup.md) for automation.*