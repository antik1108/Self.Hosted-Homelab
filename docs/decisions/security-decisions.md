# Security Decision Records (SDRs)

## SDR-001: Default-Deny Firewall Posture
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Both OpenWrt and Proxmox exposed to potential threats (college Wi-Fi peers, internet via WWAN).
**Decision**: Default-deny on all ingress; explicit allow only for required management traffic.
**Implementation**:
- OpenWrt: WAN zone `input: REJECT`, `forward: DROP`; LAN zone `input: ACCEPT`
- Proxmox: pve-firewall `policy_in: DROP`; explicit rules for SSH (22), Web UI (8006), SPICE (3128)
**Residual Risk**: Legitimate traffic blocked if rules incomplete; mitigated by testing simulations.

## SDR-002: Management Interface Binding
**Date**: 2026-11-12
**Status**: Partially Implemented
**Context**: Services listening on all interfaces increase attack surface.
**Decision**: Bind management services to LAN interface only where possible.
**Implementation**:
- ✅ OpenWrt dropbear: `Interface='lan'` (192.168.1.1:22 only)
- ✅ OpenWrt uhttpd: `listen_http='192.168.1.1:80'`, `listen_https='192.168.1.1:443'`
- ⚠️ Proxmox pveproxy: `LISTEN_IP="*"` (all interfaces) - restricted by firewall instead
- ⚠️ Proxmox sshd: `ListenAddress` not set - restricted by firewall instead
**Rationale for Proxmox**: pveproxy `LISTEN_IP` supports single IP only; firewall provides equivalent control.

## SDR-003: SSH Key-Only Authentication
**Date**: 2026-11-12
**Status**: Partially Implemented
**Context**: Password authentication vulnerable to brute force.
**Decision**: Enforce public key authentication; disable passwords.
**Implementation**:
- ✅ OpenWrt dropbear: `PasswordAuth='off'`, `RootPasswordAuth='off'`
- ⚠️ Proxmox sshd: Password auth still enabled (Ed25519 key primary)
**Reason for Proxmox Exception**: Tailscale + firewall restrict SSH to known IPs; key rotation easier via Ansible; fallback for emergency.

## SDR-004: HTTPS Enforcement with Redirect
**Date**: 2026-11-12
**Status**: Implemented
**Context**: HTTP credentials and session cookies exposed on LAN.
**Decision**: Redirect all HTTP to HTTPS (307); disable plain HTTP access.
**Implementation**:
- OpenWrt uhttpd: `redirect_https='1'` (HTTP 80 → HTTPS 443, 307)
- Proxmox pveproxy: HTTPS only (port 8006)
**Certificate**: Self-signed EC P-256, 397-day validity (browser warning accepted).

## SDR-005: Regulatory Domain Compliance (India)
**Date**: 2026-11-12
**Status**: Implemented
**Context**: Operating wireless in India requires compliance with WPC/DoT regulations.
**Decision**: Set `country='IN'` on both radios (radio0 2.4GHz, radio1 5GHz).
**Impact**:
- Enables DFS channels (52-144) with radar detection
- Restricts transmit power to Indian limits
- Channel 149 (5745 MHz) allowed for STA (non-DFS)
- Compliance with local law; avoids interference penalties.

## SDR-006: 5GHz Access Point Disabled
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Single 5GHz radio (phy1) used for WWAN STA on channel 149 (upstream college Wi-Fi).
**Decision**: Disable 5GHz AP to maintain stable upstream connection.
**Trade-off**:
- ✅ Stable WWAN (primary internet)
- ✅ No channel switching disconnections
- ❌ No 5GHz AP for LAN clients (2.4GHz only)
**Future**: Router replacement with dual 5GHz radios (see HWD-001).

## SDR-007: rpcbind Removal
**Date**: 2026-11-12
**Status**: Implemented
**Context**: rpcbind (portmapper) on port 111 exposed unnecessary attack surface; no NFS in use.
**Decision**: Stop, disable, mask rpcbind.socket and rpcbind service.
**Verification**: `ss -tulpn | grep 111` returns no output.

## SDR-008: Tailscale as Trusted Overlay
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Need secure remote access without port forwarding.
**Decision**: Tailscale WireGuard mesh for all remote management; `ts-input` chain handles Tailscale traffic before host firewall.
**Controls**:
- Tailscale ACLs in admin panel (tag-based)
- Proxmox firewall allows Tailscale IP (100.122.224.81/32) as supplementary
- DERP relay (Mumbai) for NAT traversal
- No exit node / subnet router configured
**Residual Risk**: Tailscale control plane compromise; mitigated by 2FA on Tailscale account.

## SDR-009: Ansible Vault for Secrets
**Date**: 2026-11-12
**Status**: Implemented
**Context**: WiFi passwords, SSH keys, Proxmox password must not be in plaintext.
**Decision**: Single `secrets/vault.yml` encrypted with AES256; password in `secrets/vault_pass.txt` (600, gitignored).
**Variables Protected**:
- `vault_wifi_2g_password`
- `vault_wifi_5g_password`
- `vault_ssh_public_key` (Ed25519 for Proxmox, RSA for OpenWrt via Ansible)
- `vault_proxmox_password`
**Backup**: Vault password in password manager (1Password/Bitwarden).

## SDR-010: No Public Services / Port Forwarding
**Date**: 2026-11-12
**Status**: Accepted
**Context**: Double NAT (college → OpenWrt) + CGNAT makes port forwarding unreliable and risky.
**Decision**: Zero inbound port forwarding; all remote access via Tailscale.
**Impact**:
- Cannot host public websites, game servers, etc.
- All access requires Tailscale client + authentication
- Simplified firewall (no DNAT rules)

## SDR-011: Self-Signed Certificates for Internal Services
**Date**: 2026-11-12
**Status**: Accepted
**Context**: LuCI (OpenWrt) and Proxmox Web UI need TLS but no public CA for `.lan` domains.
**Decision**: Self-signed EC P-256 certificates (397-day validity).
**Mitigation**:
- Verify SHA256 fingerprint on first connect
- Store fingerprints in documentation
- Not used for public-facing services
**Alternative Considered**: mkcert local CA - rejected as overkill for 2 services.

## SDR-012: Overlay Space Monitoring (OpenWrt)
**Date**: 2026-11-12
**Status**: Monitoring
**Context**: 960KB overlay, 93% used (72KB free); no opkg, cannot install packages.
**Decision**: Weekly `df -h /overlay` check; clean ROM duplicates if needed; no package installs.
**Threshold**: Alert if <50KB free.
**ROM Duplicate Cleanup** (safe, idempotent):
```bash
rm -rf /overlay/upper/www/luci-static \
       /overlay/upper/usr/bin/ucode \
       /overlay/upper/usr/lib/libucode.so* \
       /overlay/upper/usr/share/ucode \
       /overlay/upper/usr/share/luci \
       /overlay/upper/usr/share/rpcd \
       /overlay/upper/usr/share/acl.d \
       /overlay/upper/usr/share/ucitrack
```

## SDR-013: Backup Encryption
**Date**: 2026-11-12
**Status**: Not Implemented (Deferred)
**Context**: Backups contain SSH keys, TLS certs, config with secrets references.
**Decision**: Backups stored unencrypted locally; rely on filesystem permissions and physical security.
**Future**: Encrypt backup archives with age/gpg before offsite sync.

## SDR-014: Firmware Update Policy
**Date**: 2026-11-12
**Status**: Manual Quarterly
**Context**: OpenWrt snapshots and Proxmox VE have frequent security updates.
**Decision**: 
- OpenWrt: Check releases monthly; sysupgrade quarterly (backup first)
- Proxmox: `apt update && apt list --upgradable` monthly; apply security updates quarterly
- Tailscale: Auto-updates via package manager
**Verification**: Post-update connectivity test (`ansible-playbook playbooks/verify.yml`)

## SDR-015: Incident Response Procedures
**Date**: 2026-11-12
**Status**: Documented
**Context**: Need defined steps for suspected compromise.
**Procedures Documented**:
- OpenWrt compromise: Isolate → Verify logs → Restore backup → Rotate keys
- Proxmox compromise: Isolate (pve-firewall stop) → Verify → Reinstall → Restore /etc/pve
- Lost control machine: Revoke Tailscale key → Rotate SSH keys → Recover vault from password manager
**Testing**: Semi-annual tabletop exercise.

## SDR-016: No IPv6 Internet Exposure
**Date**: 2026-11-12
**Status**: Accepted
**Context**: No IPv6 prefix from upstream; ULA only.
**Decision**: No IPv6 firewall rules for internet; ULA traffic only on LAN.
**Impact**: Simplified IPv6 posture; no accidental IPv6 exposure.

---

## Risk Register

| ID | Risk | Likelihood | Impact | Mitigation | Status |
|----|------|------------|--------|------------|--------|
| R1 | OpenWrt overlay exhaustion | High | Medium | Weekly monitoring, ROM cleanup | Monitoring |
| R2 | Tailscale DERP latency | Medium | Low | Accept; upgrade to direct when possible | Accepted |
| R3 | Self-signed cert MITM | Low | Medium | Fingerprint verification on first connect | Mitigated |
| R4 | Proxmox SSH password brute force | Low | High | Tailscale + firewall restrict to 2 IPs | Mitigated |
| R5 | College Wi-Fi MITM on WWAN | Medium | High | HTTPS everywhere; verify certs; no sensitive cleartext | Mitigated |
| R6 | Lost vault password | Low | Critical | Store in password manager | Mitigated |
| R7 | Hardware failure (router) | Medium | High | Backup configs; router replacement plan | Documented |
| R8 | Hardware failure (Proxmox) | Low | High | Backup configs + VM backups; replacement plan | Documented |

---

*See [Architecture Decisions](architecture-decisions.md) for structural ADRs, [Security Architecture](../architecture/security-architecture.md) for controls, [Phase 4 Report](../phases/phase-4.md) for implementation.*