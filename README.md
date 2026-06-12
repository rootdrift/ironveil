# ironveil

Hardened Fedora workstation built to a defence-in-depth security model. Full-disk encryption
with hardware-key unlock, remote SSH decryption from a GrapheneOS mobile platform, WireGuard
VPN with DNS filtering, and auditability at every layer. Living build — actively maintained.

---

## Overview

Most workstation hardening stops at disk encryption. IRONVEIL goes further: the LUKS2 volume
cannot be unlocked without a physical hardware key, and if the key is unavailable the machine
can still be unlocked remotely from a hardened mobile platform over SSH — without the initramfs
ever trusting a password over the network.

The build addresses a specific threat model for a security practitioner whose workstation stores
research, tooling, and keys that are worth protecting to a standard that survives physical access,
supply-chain key compromise, and remote theft scenarios. Each component was chosen because it
eliminates a specific attack path — not because it adds capability.

---

## Threat Model

**Adversary assumptions:**

- Physical access to the running or powered-off machine (theft, border crossing, hostile
  seizure)
- Capability to clone disk and attempt offline key derivation
- Network-level adversary capable of observing traffic and injecting packets
- Credential theft from software key stores or password managers

**Design responses:**

| Threat | Control |
|--------|---------|
| Offline disk clone and brute-force | LUKS2 with Argon2id KDF; hardware key required to unlock |
| Primary hardware key lost or seized | NK#2 backup keyslot; passphrase emergency fallback |
| Hardware key cloned without touch interaction | Touch-only FIDO2 enrollment; Nitrokey 3A NFC requires physical presence |
| Remote workstation inaccessible for unlock | dracut-sshd: SSH into initramfs from GrapheneOS |
| DNS traffic leakage and exfiltration | AdGuard Home filtering; systemd-resolved bound to loopback |
| Traffic interception and geolocation | WireGuard VPN on all external traffic |

---

## Components

### LUKS2 Full-Disk Encryption

Encrypted volume: LUKS2 container UUID `6cbc50ba-6f8a-4932-abfc-f2d0504a29b3`, mapped as
`luks-6cbc50ba-6f8a-4932-abfc-f2d0504a29b3` (`aes-xts-plain64`, 512-bit key).

Three keyslots configured:

| Slot | Type | Purpose |
|------|------|---------|
| 0 | Passphrase | Emergency fallback only — long, stored offline |
| 1 | Nitrokey NK#1 (primary) | Daily driver — touch required for unlock |
| 2 | Nitrokey NK#2 (backup) | Kept offline; activated only if NK#1 is lost |

LUKS2 uses Argon2id as the key derivation function. The hardware key slots enroll FIDO2
credentials — the LUKS2 passphrase is never sent to the key; the key produces a credential
that is combined with a stored key file to derive the slot key.

### Nitrokey 3A NFC FIDO2 Hardware Key Enrollment

- **Hardware:** Nitrokey 3A NFC
- **Firmware:** 1.8.3
- **Enrollment mode:** Touch-only — physical presence required for every unlock event
- **clientPin:** Not supported on this firmware version; reliance is on touch confirmation
  rather than PIN-gated credentials
- **FIDO2 resident keys:** Used for workstation unlock; not shared with web authentication flows

The touch-only constraint is intentional. A key that could be activated remotely (via
clientPin over USB) would defeat the physical-presence guarantee. Firmware 1.8.3 does not
expose clientPin, which closes that attack path by default.

### dracut-sshd — Remote Unlock via SSH

The initramfs is built with `dracut-sshd`, which starts a minimal SSH daemon at pre-boot,
before the LUKS2 volume is mounted.

**Unlock flow:**

1. Machine powers on and reaches the initramfs SSH listener
2. GrapheneOS (Termux) connects via `ssh unlock@$IRONVEIL_IP`
3. The Termux ed25519 public key is baked into the initramfs authorized\_keys at build time
4. Authenticated session drives the LUKS2 unlock over the encrypted channel (the initramfs
   `fido2` module lets a Nitrokey touch satisfy the prompt; `systemd-tty-ask-password-agent`
   relays it)
5. Volume unlocks; boot continues to userspace

The ed25519 key baked into the initramfs is separate from all other SSH keys and is rotated
when the initramfs is rebuilt. The connection is encrypted: the initramfs SSH host key is
pinned on the GrapheneOS client to prevent MITM substitution during unlock.

**Security properties of this design:**

- The LUKS2 passphrase is only ever transmitted inside an encrypted SSH session from a
  trusted, hardware-attested device
- The unlock path requires the physical GrapheneOS device (something you have) plus the
  Termux key file (something you have, separately)
- An attacker with network access but not the GrapheneOS device cannot complete the unlock

### WireGuard VPN — wg-CH-FI-2 and wg-SE-FI-1

- **Interfaces:** two NetworkManager tunnels, `wg-CH-FI-2` and `wg-SE-FI-1` (named-tunnel model,
  replacing the legacy `wg0`); the identifier encodes the endpoint region for multi-tunnel
  readability
- **Management:** NetworkManager — each tunnel is a first-class connection. Activation is
  **manual** (`autoconnect=false`)
- **Routing:** both carry full-tunnel `AllowedIPs` (`0.0.0.0/0, ::/0`), so while one is up it is
  the default route for all traffic — a *route-based* (implicit) kill-switch rather than a separate
  fail-closed rule (see [hardening/network-stack.md](hardening/network-stack.md) for the honest
  caveat and the planned hard kill-switch)

### AdGuard Home DNS Filtering

- **Upstream DNS:** **Quad9 over DNS-over-HTTPS** (`https://dns10.quad9.net/dns-query`) — every
  query leaves AdGuard already encrypted, then egresses through the active WireGuard tunnel. (The
  VPN provider also pushes `10.2.0.1`, but AdGuard overrides it.)
- **Listener:** `*:53` (pid 1452) — AdGuard is the system resolver. The no-plaintext-egress
  property comes from the DoH-over-tunnel upstream, not from a loopback bind.
- **Block list:** AdGuard DNS filter (`filter_1.txt`) enabled
- **systemd-resolved:** forwards all queries to `127.0.0.1` (stub listener off) — AdGuard owns
  port 53 and intercepts before anything reaches the upstream

The DNS chain: application → systemd-resolved (127.0.0.1) → AdGuard Home (:53) → Quad9 DoH
(encrypted) → egress via the active WireGuard tunnel. An attacker observing the external interface
sees only WireGuard-encrypted traffic; the upstream resolver sees the WireGuard exit IP, not the
host IP, and never a plaintext query.

### OpenRGB Peripheral Configuration

- **Razer Huntsman V2** (keyboard)
- **Razer Basilisk** (mouse)
- **6× Corsair fans via Commander Pro** (chassis)

OpenRGB provides a unified, vendor-agnostic interface for RGB configuration without requiring
vendor cloud daemons (Razer Synapse, iCUE). This matters for a hardened workstation: vendor
software of this class commonly phones home, requires account registration, and has historically
contained telemetry that cannot be disabled without network-level blocking.

The Commander Pro is detected as a USB HID device. All six fan channels are configured via
OpenRGB's profile system — no Corsair software is installed.

Full configuration — channel layout, the 96-LED Commander Pro fan chain, udev rules for
non-root HID access, and verification commands — is documented in
[hardening/openrgb-setup.md](hardening/openrgb-setup.md).

<!-- MANUAL INPUT REQUIRED: record the saved OpenRGB profile name and colour scheme from the OpenRGB GUI -->
<!-- OpenRGB udev rule: RESOLVED 2026-06-12 — OpenRGB 0.9+ (git) installed; HID access rule at /usr/lib/udev/rules.d/60-openrgb.rules (confirmed present). Saved profile NAME still to capture from the GUI (line above). -->

---

## Security Architecture — Defence in Depth

The build treats each layer as independently exploitable and compensating controls at every
layer:

```
Physical layer:   Nitrokey touch-only FIDO2 — presence required
Disk layer:       LUKS2 Argon2id — offline brute-force impractical
Boot layer:       dracut-sshd — remote unlock without exposing passphrase
Network layer:    WireGuard — encrypted egress; kill-switch on drop
DNS layer:        AdGuard + systemd-resolved — no plaintext queries
```

| Layer | Mechanism | Threat mitigated |
|-------|-----------|------------------|
| Physical | Nitrokey 3A NFC FIDO2, touch-only enrollment (firmware 1.8.3, no clientPin) | Remote/unattended hardware-key activation; unlock without physical presence |
| Disk | LUKS2 `aes-xts-plain64`/512-bit (UUID `6cbc50ba-…`) with Argon2id passphrase slot | Offline disk clone and brute-force key derivation |
| Boot | dracut-sshd pre-boot SSH with pinned ed25519 host key | Unattended unlock without exposing the passphrase; MITM key substitution during unlock |
| Key custody | Primary Nitrokey slot, backup Nitrokey offline slot, offline emergency passphrase | Single point of key failure; primary key loss or seizure |
| Network | WireGuard `wg-CH-FI-2` / `wg-SE-FI-1` (NetworkManager, manual) with full-tunnel routing | Traffic interception, geolocation, and leakage if the tunnel drops |
| DNS | AdGuard Home on `*:53` → Quad9 DoH over the active tunnel; systemd-resolved forwards to it | Plaintext DNS leakage; tracker, telemetry, and known-malicious domain resolution |

The hardware key is the root of the trust chain. Without a Nitrokey or the emergency
passphrase, the disk does not open. The emergency passphrase is stored offline and is
not present on the machine in any form.

### Host Hardening — MAC, seccomp, boot integrity

Above the disk and network layers, the host runs **SELinux in `enforcing`/`targeted` mode**
(deliberately not disabled), with **seccomp BPF filtering compiled into the kernel**
(`CONFIG_SECCOMP_FILTER=y`) for systemd service sandboxing. Two controls are documented as
**honest gaps** rather than hidden: Yama `ptrace_scope` is at the Fedora default `0` (tightening
to `1` is planned), and **UEFI Secure Boot is currently disabled / in Setup Mode** — the build's
unlock-integrity guarantee rests on LUKS2 + touch-only FIDO2 + a pinned initramfs host key, not on
Secure Boot. Full detail, command output, and the planned next moves (ptrace tightening; Secure
Boot + TPM2 PCR sealing to close the evil-maid gap) are in
[hardening/os-hardening.md](hardening/os-hardening.md).

> Measured Boot note: the current build does not bind the LUKS unlock to TPM PCR measurements
> (see [hardening/research/luks2-fido2-reference.md](hardening/research/luks2-fido2-reference.md)
> §5). This is tracked as FUTURE WORK alongside the Secure Boot enrolment above.

---

## Results and Current State

| Component | Status |
|-----------|--------|
| LUKS2 `aes-xts-plain64`/512-bit | Operational — 3 keyslots (Argon2id passphrase + 2× FIDO2) |
| Nitrokey 3A NFC FIDO2 (primary) | Operational — firmware 1.8.3, touch-only, no clientPin |
| Nitrokey 3A NFC backup keyslot | Enrolled — stored offline |
| dracut-sshd remote unlock | Operational — v0.7.1-5.fc44; systemd-networkd + fido2 modules |
| WireGuard wg-CH-FI-2 / wg-SE-FI-1 | Operational — NetworkManager, manual, full-tunnel |
| AdGuard Home | Operational — Quad9 DoH upstream, `*:53`, AdGuard DNS filter |
| systemd-resolved → 127.0.0.1 | Operational |
| Build platform | Fedora 44, kernel `7.0.11-200.fc44.x86_64` |
| SELinux | Enforcing — `targeted` policy, MLS compiled |
| seccomp | Available — `CONFIG_SECCOMP_FILTER=y` (systemd sandboxing) |
| UEFI Secure Boot | Disabled / Setup Mode — honest gap, FUTURE WORK |
| OpenRGB (Razer + Corsair) | Operational — vendor daemons absent (udev `60-openrgb.rules`) |

<!-- MANUAL INPUT REQUIRED: add LUKS2 unlock-latency benchmark data (hardware key vs passphrase) measured on the running machine — not captured in the 2026-06-11 hardware session -->
<!-- Fedora release + kernel: RESOLVED 2026-06-11 (Fedora 44, 7.0.11-200.fc44.x86_64 — see Build platform row above). -->
<!-- dracut-sshd build + config drop-in: RESOLVED 2026-06-11 — documented in hardening/dracut-sshd.md (v0.7.1-5.fc44; systemd-networkd + fido2 modules; rebuild via `sudo dracut -f --regenerate-all`). -->

---

## Skills Demonstrated

| Skill area | Evidence |
|------------|---------|
| Linux hardening | LUKS2 full-disk encryption; keyslot management; Argon2id KDF |
| FIDO2 / hardware key integration | Nitrokey 3A NFC enrolled as LUKS2 keyslot; touch-only enforcement |
| Initramfs engineering | dracut-sshd configured for pre-boot SSH with pinned ed25519 key |
| Network security | WireGuard VPN with kill-switch; NetworkManager integration |
| DNS security | AdGuard Home + systemd-resolved; DNS-over-WireGuard; loopback binding |
| Defence in depth | Each threat mapped to a compensating control; no single point of failure |
| Operational security | Vendor cloud daemons eliminated; RGB via OpenRGB without telemetry |
| Remote operations | SSH-based unlock from GrapheneOS; no physical presence required for boot |

---

*Part of the [rootdrifter](https://github.com/rootdrifter) security portfolio — built and maintained by a security-cleared candidate. UK-issued clearance held now, not pending vetting: deployable to cleared work from day one.*
