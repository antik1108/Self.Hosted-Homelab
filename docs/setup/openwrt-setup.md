# OpenWrt Router Setup

## Installation

### 1. Verify Device Support
- Check [OpenWrt Table of Hardware](https://openwrt.org/toh/start) for your device
- This project uses: **TP-Link Archer C20 v5** (ramips/mt76x8)
- Confirm: Flash ≥16 MB, RAM ≥64 MB

### 2. Download Firmware
- **Stable**: [downloads.openwrt.org](https://downloads.openwrt.org/releases/)
- **Snapshot**: [downloads.openwrt.org/snapshots/](https://downloads.openwrt.org/snapshots/)
- For Archer C20 v5: `openwrt-ramips-mt76x8-tplink_archer-c20-v5-squashfs-sysupgrade.bin`

### 3. Flash OpenWrt

#### Option A: TFTP Recovery (Most Devices)
```bash
# 1. Set static IP on computer: 192.168.1.100/24
# 2. Connect Ethernet to LAN port 1
# 3. Power off router
# 4. Hold reset button, power on, hold until LED flashes
# 5. TFTP transfer:
tftp 192.168.1.1
binary
put openwrt-ramips-mt76x8-tplink_archer-c20-v5-squashfs-sysupgrade.bin
```

#### Option B: Vendor Web UI Upgrade
1. Login to stock firmware web UI
2. System → Firmware Upgrade
3. Select OpenWrt sysupgrade image
4. Upgrade and wait

#### Option C: Sysupgrade from Existing OpenWrt
```bash
scp openwrt-*.bin root@192.168.1.1:/tmp/
ssh root@192.168.1.1 "sysupgrade -v /tmp/openwrt-*.bin"
```

---

## Initial Configuration

### 1. First Boot
```bash
# Connect via Ethernet to LAN port
# Default: 192.168.1.1, no password
ssh root@192.168.1.1
```

### 2. Set Root Password
```bash
passwd
# Enter strong password (store in vault)
```

### 3. Configure SSH Keys (Recommended)
```bash
# On control machine
cat ~/.ssh/id_rsa.pub | ssh root@192.168.1.1 "mkdir -p /etc/dropbear && cat >> /etc/dropbear/authorized_keys"

# Test
ssh -i ~/.ssh/id_rsa root@192.168.1.1
```

---

## Network Configuration (UCI)

### LAN Interface
```bash
uci set network.lan=interface
uci set network.lan.device='br-lan'
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.1.1'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.ip6assign='60'
```

### WAN Interface (Ethernet)
```bash
uci set network.wan=interface
uci set network.wan.device='eth0.2'
uci set network.wan.proto='dhcp'

uci set network.wan6=interface
uci set network.wan6.device='eth0.2'
uci set network.wan6.proto='dhcpv6'
```

### WWAN Interface (Wi-Fi Client) - Current Setup
```bash
# Create wireless STA interface
uci set network.wwan=interface
uci set network.wwan.proto='dhcp'
uci set network.wwan.device='phy1-sta0'

# Add wwan to WAN firewall zone
uci add_list firewall.wan.network='wwan'
```

### VLAN Configuration (mt7530)
```bash
# VLAN 1: LAN (ports 1-4, CPU tagged)
uci set network.@switch[0].vlan[1].ports='1 2 3 4 6t'

# VLAN 2: WAN (port 0, CPU tagged)
uci set network.@switch[0].vlan[2].ports='0 6t'
```

### Apply Network Changes
```bash
uci commit network
/etc/init.d/network reload
```

---

## Wireless Configuration

### 2.4 GHz AP (radio0)
```bash
uci set wireless.radio0=wifi-device
uci set wireless.radio0.type='mac80211'
uci set wireless.radio0.path='platform/10300000.wmac'
uci set wireless.radio0.channel='1'
uci set wireless.radio0.htmode='HT20'
uci set wireless.radio0.country='IN'
uci set wireless.radio0.cell_density='0'

uci set wireless.default_radio0=wifi-iface
uci set wireless.default_radio0.device='radio0'
uci set wireless.default_radio0.network='lan'
uci set wireless.default_radio0.mode='ap'
uci set wireless.default_radio0.ssid='HomeLab-2G'
uci set wireless.default_radio0.encryption='psk2+ccmp'
uci set wireless.default_radio0.key='{{ vault_wifi_2g_password }}'
```

### 5 GHz STA (radio1 - Upstream Client)
```bash
uci set wireless.radio1=wifi-device
uci set wireless.radio1.type='mac80211'
uci set wireless.radio1.path='pci0000:00/0000:00:00.0/0000:01:00.0'
uci set wireless.radio1.channel='149'
uci set wireless.radio1.htmode='VHT40'
uci set wireless.radio1.country='IN'
uci set wireless.radio1.cell_density='0'

uci set wireless.wwan=wifi-iface
uci set wireless.wwan.device='radio1'
uci set wireless.wwan.network='wwan'
uci set wireless.wwan.mode='sta'
uci set wireless.wwan.ssid='Rishihood_Learner'
uci set wireless.wwan.encryption='psk2'
uci set wireless.wwan.key='{{ vault_wifi_5g_password }}'
```

### 5 GHz AP (radio1 - DISABLED Due to Channel Conflict)
```bash
# Hardware limitation: Single radio cannot do STA+AP on different channels
uci set wireless.default_radio1=wifi-iface
uci set wireless.default_radio1.device='radio1'
uci set wireless.default_radio1.network='lan'
uci set wireless.default_radio1.mode='ap'
uci set wireless.default_radio1.ssid='HomeLab-5G'
uci set wireless.default_radio1.encryption='sae-mixed'
uci set wireless.default_radio1.key='{{ vault_wifi_5g_password }}'
uci set wireless.default_radio1.disabled='1'  # DISABLED
```

### Apply Wireless Changes
```bash
uci commit wireless
wifi reload
```

---

## DHCP & DNS (dnsmasq)

### DHCP Server
```bash
uci set dhcp.lan=dhcp
uci set dhcp.lan.interface='lan'
uci set dhcp.lan.start='100'
uci set dhcp.lan.limit='150'
uci set dhcp.lan.leasetime='12h'
uci set dhcp.lan.dhcpv6='server'
uci set dhcp.lan.ra='server'
uci set dhcp.lan.ra_slaac='1'

# Static reservations (MAC → IP)
uci add dhcp host
uci set dhcp.@host[-1].name='proxmox-host'
uci set dhcp.@host[-1].mac='00:e0:4c:68:5e:a8'
uci set dhcp.@host[-1].ip='192.168.1.10'

uci add dhcp host
uci set dhcp.@host[-1].name='macbook'
uci set dhcp.@host[-1].mac='34:66:91:ef:4e:2d'
uci set dhcp.@host[-1].ip='192.168.1.50'
```

### DNS Configuration
```bash
uci set dhcp.@dnsmasq[0].domain='lan'
uci set dhcp.@dnsmasq[0].local='/lan/'
uci set dhcp.@dnsmasq[0].expandhosts='1'
uci set dhcp.@dnsmasq[0].authoritative='1'
uci set dhcp.@dnsmasq[0].readethers='1'
uci set dhcp.@dnsmasq[0].leasefile='/tmp/dhcp.leases'
uci set dhcp.@dnsmasq[0].resolvfile='/tmp/resolv.conf.d/resolv.conf.auto'
uci set dhcp.@dnsmasq[0].nonegcache='0'
uci set dhcp.@dnsmasq[0].cachesize='1000'
uci set dhcp.@dnsmasq[0].localise_queries='1'
uci set dhcp.@dnsmasq[0].rebind_protection='1'
uci set dhcp.@dnsmasq[0].rebind_localhost='1'
uci set dhcp.@dnsmasq[0].filterwin2k='0'
uci set dhcp.@dnsmasq[0].localservice='1'       # Only serve LAN
uci set dhcp.@dnsmasq[0].nonwildcard='1'        # Bind to specific interfaces
uci set dhcp.@dnsmasq[0].dnssec='1'
uci set dhcp.@dnsmasq[0].ednspacket_max='1232'

# Upstream DNS servers
uci add_list dhcp.@dnsmasq[0].server='1.1.1.1'
uci add_list dhcp.@dnsmasq[0].server='8.8.8.8'
uci add_list dhcp.@dnsmasq[0].server='9.9.9.9'
```

### Apply DHCP/DNS Changes
```bash
uci commit dhcp
/etc/init.d/dnsmasq reload
```

---

## Firewall Configuration (fw4/nftables)

### Default Policies
```bash
uci set firewall.@defaults[0]=defaults
uci set firewall.@defaults[0].syn_flood='1'
uci set firewall.@defaults[0].input='REJECT'
uci set firewall.@defaults[0].output='ACCEPT'
uci set firewall.@defaults[0].forward='REJECT'
```

### LAN Zone
```bash
uci set firewall.lan=zone
uci set firewall.lan.name='lan'
uci set firewall.lan.input='ACCEPT'
uci set firewall.lan.output='ACCEPT'
uci set firewall.lan.forward='ACCEPT'
uci set firewall.lan.network='lan'
```

### WAN Zone (includes wwan)
```bash
uci set firewall.wan=zone
uci set firewall.wan.name='wan'
uci set firewall.wan.input='REJECT'
uci set firewall.wan.output='ACCEPT'
uci set firewall.wan.forward='DROP'
uci set firewall.wan.masq='1'
uci set firewall.wan.mtu_fix='1'
uci set firewall.wan.network='wan wan6 wwan'
```

### LAN → WAN Forwarding
```bash
uci set firewall.lan_wan=forwarding
uci set firewall.lan_wan.src='lan'
uci set firewall.lan_wan.dest='wan'
```

### Essential WAN Rules
```bash
# DHCP renew
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-DHCP-Renew'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='68'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='ipv4'

# ICMP
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Ping'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='icmp'
uci set firewall.@rule[-1].icmp_type='echo-request'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='ipv4'

# IGMP
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-IGMP'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='igmp'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='ipv4'

# DHCPv6
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-DHCPv6'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='546'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='ipv6'

# ICMPv6
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-ICMPv6'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='icmp'
uci set firewall.@rule[-1].icmp_type='echo-request echo-reply destination-unreachable packet-too-big time-exceeded bad-header unknown-header-type router-solicitation neighbour-solicitation router-advertisement neighbour-advertisement'
uci set firewall.@rule[-1].limit='1000/sec'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='ipv6'
```

### Apply Firewall Changes
```bash
uci commit firewall
/etc/init.d/firewall reload
```

---

## SSH (dropbear) Hardening

### Bind to LAN Only
```bash
uci set dropbear.@dropbear[0].Interface='lan'
```

### Key-Only Authentication
```bash
uci set dropbear.@dropbear[0].PasswordAuth='off'
uci set dropbear.@dropbear[0].RootPasswordAuth='off'
uci set dropbear.@dropbear[0].Port='22'
uci set dropbear.@dropbear[0].RSAAuth='on'
uci set dropbear.@dropbear[0].PubkeyAuth='on'
```

### Apply SSH Changes
```bash
uci commit dropbear
/etc/init.d/dropbear reload
```

### Verify
```bash
# Should work (key)
ssh -i ~/.ssh/id_rsa root@192.168.1.1

# Should fail (password)
ssh -o PreferredAuthentications=password root@192.168.1.1
```

---

## LuCI (uhttpd) Hardening

### Bind to LAN Only
```bash
uci set uhttpd.main.listen_http='192.168.1.1:80'
uci set uhttpd.main.listen_https='192.168.1.1:443'
```

### Enable HTTPS Redirect
```bash
uci set uhttpd.main.redirect_https='1'
```

### Apply LuCI Changes
```bash
uci commit uhttpd
/etc/init.d/uhttpd reload
```

### Verify
```bash
# HTTP should redirect
curl -k http://192.168.1.1
# Should return 307 to HTTPS

# HTTPS should work
curl -k https://192.168.1.1
# Should return 200
```

---

## System Settings

### Timezone (Asia/Kolkata)
```bash
uci set system.@system[0].timezone='IST-5:30'
uci set system.@system[0].zonename='Asia/Kolkata'
uci commit system
/etc/init.d/sysntpd restart
```

### NTP Client
```bash
uci set system.ntp=timeserver
uci set system.ntp.enabled='1'
uci set system.ntp.enable_server='0'
uci add_list system.ntp.server='0.openwrt.pool.ntp.org'
uci add_list system.ntp.server='1.openwrt.pool.ntp.org'
uci add_list system.ntp.server='2.openwrt.pool.ntp.org'
uci add_list system.ntp.server='3.openwrt.pool.ntp.org'
uci commit system
/etc/init.d/sysntpd restart
```

### Hostname
```bash
uci set system.@system[0].hostname='main-router'
uci commit system
```

---

## Regulatory Domain (India)

### Apply to Both Radios
```bash
uci set wireless.radio0.country='IN'
uci set wireless.radio1.country='IN'
uci commit wireless
wifi reload
```

### Verify
```bash
iw phy phy0 info | grep -A 20 'Frequencies:'
iw phy phy1 info | grep -A 20 'Frequencies:'
# Should show channels with "radar detection" for DFS channels
```

---

## Backup & Restore

### Create Sysupgrade Backup
```bash
sysupgrade -b /tmp/openwrt-backup-$(date -Iseconds).tar.gz
scp root@192.168.1.1:/tmp/openwrt-backup-*.tar.gz ./backups/
```

### Create UCI Config Dump
```bash
ssh root@192.168.1.1 "uci show" > backups/openwrt/uci-configs.txt
```

### Restore from Sysupgrade Backup
```bash
scp openwrt-backup-*.tar.gz root@192.168.1.1:/tmp/
ssh root@192.168.1.1 "sysupgrade -r /tmp/openwrt-backup-*.tar.gz && reboot"
```

### Restore Individual Configs
```bash
# Via Ansible (recommended)
ansible-playbook playbooks/openwrt.yml --vault-password-file secrets/vault_pass.txt
```

---

## Overlay Space Management

### Check Usage
```bash
df -h /overlay
# Should show ~960K total, ~72K free (93% used)
```

### Clean Duplicate ROM Files (If Needed)
```bash
# Check what's in overlay
du -h /overlay/upper/* | sort -h

# Remove duplicates of ROM files (safe - falls back to /rom)
rm -rf /overlay/upper/www/luci-static \
       /overlay/upper/usr/bin/ucode \
       /overlay/upper/usr/lib/libucode.so* \
       /overlay/upper/usr/share/ucode \
       /overlay/upper/usr/share/luci \
       /overlay/upper/usr/share/rpcd \
       /overlay/upper/usr/share/acl.d \
       /overlay/upper/usr/share/ucitrack
```

> **Warning**: Only remove files that exist in `/rom/`. Never remove `/overlay/upper/etc/` (your configs).

---

## Verification Checklist

- [ ] OpenWrt accessible at `http://192.168.1.1` (redirects to HTTPS)
- [ ] SSH key authentication working, password auth disabled
- [ ] LAN interface: `192.168.1.1/24`, DHCP serving 100-250
- [ ] WWAN connected to `Rishihood_Learner` (10.7.28.50)
- [ ] 2.4GHz AP broadcasting `HomeLab-2G` (WPA2)
- [ ] 5GHz STA connected to upstream, 5GHz AP disabled
- [ ] Firewall: WAN zone `input: REJECT`, LAN zone `input: ACCEPT`
- [ ] LuCI accessible only on `192.168.1.1:443`, HTTP redirects
- [ ] Dropbear SSH only on `192.168.1.1:22`, key-only
- [ ] DNS: `localservice=1`, `nonwildcard=1`, upstream 1.1.1.1/8.8.8.8/9.9.9.9
- [ ] Timezone: Asia/Kolkata, NTP syncing
- [ ] Regulatory domain: IN on both radios
- [ ] Static DHCP reservations for Proxmox (1.10) and MacBook (1.50)
- [ ] Backup created and verified

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Can't access LuCI** | Check `uhttpd` listen addresses, firewall, try `https://192.168.1.1` |
| **SSH key not working** | Verify `/etc/dropbear/authorized_keys` format, check dropbear logs |
| **WWAN not connecting** | Check `wpa_supplicant` logs, verify SSID/password, radio1 country=IN |
| **5GHz AP not starting** | Hardware limitation: STA/AP must share channel; disable AP or match channel |
| **Overlay full** | Clean duplicate ROM files, avoid installing packages |
| **DHCP not working** | Check `dnsmasq` logs, verify interface binding, check firewall |
| **DNS not resolving** | Check `resolv.conf.auto`, upstream DNS servers, rebind protection |
| **Config not persisting** | Run `uci commit` after changes, check overlay space |

---

*See [Architecture Overview](../architecture/overview.md) for context, [Network Topology](../architecture/network-topology.md) for diagrams, [Prerequisites](prerequisites.md) for requirements, and [Ansible Setup](ansible-setup.md) for automation.*