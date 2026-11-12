# Security Architecture

## Threat Model

### Assets to Protect
| Asset | Classification | Impact if Compromised |
|-------|----------------|----------------------|
| Proxmox host (hypervisor) | Critical | Full VM/CT compromise, persistence, lateral movement |
| OpenWrt router (gateway) | Critical | Network interception, MITM, pivot to LAN |
| SSH keys (control machine) | Critical | Full infrastructure access |
| Ansible Vault secrets | High | WiFi passwords, Proxmox credentials |
| Tailscale identity | High | Overlay network access |
| LAN devices | Medium | Data theft, botnet recruitment |

### Threat Actors
| Actor | Capability | Motivation | Likelihood |
|-------|------------|------------|------------|
| **College network peers** | Same broadcast domain (WWAN) | Opportunistic, curiosity | High |
| **Internet attackers** | Remote, automated scans | Botnet, crypto-mining | Medium (via WWAN) |
| **Tailscale control plane compromise** | Supply chain | Targeted | Low |
| **Lost/stolen control machine** | Physical access to keys | Opportunistic | Low |
| **Insider (shared dorm)** | Physical LAN access | Curiosity, malice | Medium |

### Attack Surfaces

| Surface | Exposure | Current Mitigation |
|---------|----------|-------------------|
| **WWAN interface (10.7.28.50)** | College Wi-Fi peers | OpenWrt firewall: WAN zone `input: REJECT` |
| **Proxmox Web UI (8006)** | LAN + Tailscale | pve-firewall: allow only 192.168.1.0/24 + 100.122.224.81 |
| **Proxmox SSH (22)** | LAN + Tailscale | pve-firewall: allow only 192.168.1.0/24 + 100.122.224.81 |
| **OpenWrt SSH (22)** | LAN only | Dropbear bind to br-lan |
| **OpenWrt LuCI (443)** | LAN only | uhttpd bind to 192.168.1.1 |
| **Tailscale UDP (41641)** | Internet (DERP) | ts-input: ACCEPT from anywhere (required for DERP) |
| **DNS (53)** | LAN only | dnsmasq `localservice=1`, OpenWrt firewall blocks WAN |

---

## Defense in Depth Layers

### Layer 1: Network Segmentation
```
┌────────────────────────────────────────────────────────────────────┐
│                     UNTRUSTED: College Wi-Fi                       │
│  10.7.0.0/19  • No management  • No SSH  • No DNS  • No Web UI   │
└────────────────────────────────┬───────────────────────────────────┘
                                 │ NAT + Stateful Firewall (OpenWrt)
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│                      TRUSTED: LAN (192.168.1.0/24)                 │
│  • SSH (22)        • HTTPS (443, 8006)  • SPICE (3128)           │
│  • DHCP/DNS/NTP    • Full Proxmox mgmt  • OpenWrt mgmt           │
└────────────────────────────────┬───────────────────────────────────┘
                                 │ Tailscale Overlay (WireGuard)
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│                    TRUSTED: Tailscale (100.64.0.0/10)              │
│  • Proxmox: 100.122.224.81  • MacBook: 100.110.79.92             │
│  • SSH, Web UI, SPICE via Proxmox firewall                       │
│  • ts-input chain handles all Tailscale traffic                  │
└────────────────────────────────────────────────────────────────────┘
```

### Layer 2: Host Firewalls

#### OpenWrt (nftables/fw4)
```nft
table inet fw4 {
    chain input_wan {
        # Only these allowed from WAN/WWAN:
        udp dport 68 accept          # DHCP renew
        icmp type echo-request accept # Ping
        ip protocol igmp accept      # IGMP
        udp dport 546 accept         # DHCPv6
        icmpv6 type { mld-listener-query, ... } accept # MLD
        icmpv6 type { destination-unreachable, ... } limit rate 1000/second accept # ICMPv6
        reject                       # Everything else
    }
    
    chain forward_wan {
        icmpv6 limit rate 1000/second accept
        ip protocol esp accept       # IPsec
        udp dport 500 accept         # ISAKMP
        drop                         # Everything else
    }
}
```

**Key Rules:**
- WAN zone: `input: REJECT`, `forward: DROP`, `masq: true`
- LAN zone: `input: ACCEPT`, `forward: ACCEPT`
- Forwarding: `lan → wan` only

#### Proxmox (pve-firewall / iptables)
```
Chain INPUT (policy DROP)
    1. ts-input (Tailscale chain) - ACCEPT all from tailscale0, UDP 41641
    2. PVEFW-INPUT:
       - ACCEPT established/related
       - ACCEPT lo
       - ACCEPT SSH (22) from 192.168.1.0/24
       - ACCEPT SSH (22) from 100.122.224.81/32
       - ACCEPT Web UI (8006) from 192.168.1.0/24
       - ACCEPT Web UI (8006) from 100.122.224.81/32
       - ACCEPT SPICE (3128) from 192.168.1.0/24
       - ACCEPT SPICE (3128) from 100.122.224.81/32
       - DROP all else
```

**Management IP Set (PVEFW-0-management-v4):**
- Contains: `192.168.1.0/24`
- Default rules allow: 22, 8006, 3128, 5900-5999, 60000-60050 from this set

#### Tailscale (ts-input chain - independent of pve-firewall)
```
Chain ts-input (policy ACCEPT)
    - ACCEPT all from tailscale0 interface
    - ACCEPT UDP 41641 from anywhere (DERP relay)
    - RETURN 100.115.92.0/23 on non-tailscale0
    - DROP 100.64.0.0/10 on non-tailscale0
```

### Layer 3: Service Hardening

#### OpenWrt
| Service | Hardening Applied |
|---------|-------------------|
| **dropbear (SSH)** | Bind to `lan` interface only; key-only auth (`PasswordAuth=off`, `RootPasswordAuth=off`); RSA + Ed25519 keys |
| **uhttpd (LuCI)** | Bind to `192.168.1.1` only; HTTPS redirect enabled; self-signed TLS (EC P-256) |
| **dnsmasq** | `localservice=1`, `nonwildcard=1`, `rebind_protection=1`, DNSSEC enabled |
| **firewall** | WAN zone `input: REJECT`, `forward: DROP`; regulatory domain `IN` |
| **wireless** | 5GHz AP disabled (channel conflict); 2.4GHz WPA2; regulatory domain `IN` |
| **system** | Timezone `Asia/Kolkata` (IST-5:30); NTP client only |

#### Proxmox
| Service | Hardening Applied |
|---------|-------------------|
| **sshd** | Key-based auth (Ed25519); password auth still enabled (can disable); root login permitted |
| **pveproxy** | Listens on `*:8006` (all interfaces); restricted by pve-firewall to LAN + Tailscale IP |
| **spiceproxy** | Listens on `*:3128`; restricted by pve-firewall |
| **rpcbind** | **STOPPED, DISABLED, MASKED** (no NFS in use) |
| **pve-firewall** | **ENABLED** with `policy_in: DROP`; minimal ruleset |
| **tailscaled** | Runs on `tailscale0`; ts-input chain handles traffic |

### Layer 4: Cryptography & Authentication

| Component | Algorithm | Key Management |
|-----------|-----------|----------------|
| **SSH (Proxmox)** | Ed25519 | `~/.ssh/id_ed25519` on MacBook → `authorized_keys` on Proxmox |
| **SSH (OpenWrt)** | RSA (dropbear limitation) | `~/.ssh/id_rsa` on MacBook → `/etc/dropbear/authorized_keys` on OpenWrt |
| **Tailscale** | WireGuard (Curve25519, ChaCha20-Poly1305) | Managed by Tailscale control plane |
| **LuCI TLS** | EC P-256, ECDSA, 397-day self-signed | `/etc/uhttpd.crt`, `/etc/uhttpd.key` |
| **Ansible Vault** | AES256 | `secrets/vault_pass.txt` (600 perms) |

### Layer 5: Secrets Management

```
┌─────────────────────────────────────────────────────────────────┐
│                     ANSIBLE VAULT (AES256)                      │
│  secrets/vault.yml (encrypted)                                  │
│  secrets/vault_pass.txt (600, gitignored)                       │
│                                                                 │
│  Contains:                                                      │
│  • vault_wifi_2g_password      = "lundlemera"                   │
│  • vault_wifi_5g_password      = "HomeLab5G-SecurePass2026!"    │
│  • vault_ssh_public_key        = "ssh-ed25519 AAAA..."          │
│  • vault_proxmox_password      = "tamannahazari"                │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VARIABLE REFERENCES                          │
│  inventory/group_vars/openwrt.yml:                              │
│    wifi.radio0.key: "{{ vault_wifi_2g_password }}"              │
│    wifi.radio1.key: "{{ vault_wifi_5g_password }}"              │
│  inventory/group_vars/proxmox.yml:                              │
│    (storage tasks reference vault_proxmox_password)             │
│  roles/openwrt/tasks/ssh.yml:                                   │
│    authorized_key: "{{ vault_ssh_public_key }}"                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Security Controls Summary

| Control | OpenWrt | Proxmox | Status |
|---------|---------|---------|--------|
| **Network segmentation** | ✅ VLANs, zones | ✅ Bridge isolation | Implemented |
| **Default-deny firewall** | ✅ WAN: REJECT | ✅ policy_in: DROP | Implemented |
| **Management interface binding** | ✅ LAN only | ⚠️ All interfaces + firewall | Partial |
| **Key-only SSH** | ✅ Enforced | ⚠️ Keys + password | Partial |
| **HTTPS enforcement** | ✅ Redirect | ✅ TLS only | Implemented |
| **Regulatory compliance** | ✅ IN domain | N/A | Implemented |
| **Unnecessary services disabled** | N/A | ✅ rpcbind masked | Implemented |
| **Overlay space management** | ⚠️ 72 KB free | N/A | Monitoring |
| **Automated backups** | ✅ Manual + Ansible | ✅ Manual + Ansible | Partial |
| **Secrets encryption** | ✅ Vault | ✅ Vault | Implemented |
| **Remote access via overlay** | Via Proxmox | ✅ Tailscale | Implemented |

---

## Incident Response Procedures

### Suspected Compromise - OpenWrt
1. **Isolate**: Disconnect Ethernet from Proxmox; disable WiFi radios
2. **Verify**: Check `/var/log/messages`, `dmesg`, `netstat -tulpn`
3. **Restore**: `sysupgrade -r /tmp/backup.tar.gz` from known-good backup
4. **Rotate**: Regenerate SSH host keys, deploy new authorized_keys
5. **Audit**: Review firewall rules, UCI config for unauthorized changes

### Suspected Compromise - Proxmox
1. **Isolate**: `pve-firewall stop` (allows SSH via Tailscale for forensics) or block at Tailscale ACL
2. **Verify**: `journalctl -u sshd`, `last`, `ss -tulpn`, check for unexpected VMs/CTs
3. **Restore**: Reinstall Proxmox → restore `/etc/pve`, `/etc/network/interfaces`
4. **Rotate**: SSH host keys, Tailscale auth key, Proxmox root password
5. **Audit**: Check pve-firewall rules, storage config, cluster config

### Lost Control Machine (MacBook)
1. **Revoke**: Remove MacBook Tailscale key from admin panel
2. **Rotate**: Generate new Ed25519 key pair; deploy to Proxmox
3. **Recover**: Clone repo on new machine; restore `secrets/` from password manager
4. **Verify**: Run `ansible all -m ping` with new key

---

## Compliance & Audit Checklist

| Check | Frequency | Method | Status |
|-------|-----------|--------|--------|
| **Backup integrity** | Quarterly | `tar -tzf backup.tar.gz` | ✅ Manual |
| **Firewall rule review** | Monthly | `pve-firewall compile`, `nft list ruleset` | ✅ Manual |
| **SSH key audit** | Quarterly | `cat authorized_keys` on both hosts | ✅ Manual |
| **Vault accessibility** | Monthly | `ansible-vault view secrets/vault.yml` | ✅ Manual |
| **Overlay space** | Weekly | `df -h /overlay` on OpenWrt | ✅ Manual |
| **Tailscale device audit** | Monthly | `tailscale status` | ✅ Manual |
| **Firmware updates** | Monthly | Check OpenWrt/Proxmox releases | ⚠️ Manual |
| **Certificate expiry** | Quarterly | `openssl x509 -in /etc/uhttpd.crt -noout -dates` | ✅ Manual |

---

## Residual Risks (Accepted)

| Risk | Reason for Acceptance | Monitoring |
|------|----------------------|------------|
| **Self-signed TLS certs** | Internal lab; browser warnings acceptable | Verify fingerprint on first connect |
| **Password auth on Proxmox SSH** | Ed25519 key primary; Tailscale restricts access | Monitor auth logs |
| **OpenWrt overlay near capacity** | No opkg; minimal config changes | `df -h /overlay` weekly |
| **No 5GHz AP** | Hardware limitation (STA/AP channel conflict) | Accept until router replacement |
| **No IPv6 internet** | Upstream doesn't provide prefix delegation | ULA only for local |
| **DERP relay latency** | No direct WireGuard path (CGNAT) | Tailscale status shows relay |

---

*See [Architecture Overview](overview.md) for context, [Network Topology](network-topology.md) for diagrams, [Hardware Inventory](hardware-inventory.md) for device details, and [IP Addressing](ip-addressing.md) for allocation.*