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

Encrypted volume: `/dev/sda3`

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
4. Authenticated session allows `systemd-cryptsetup@sda3` to be triggered, supplying the
   LUKS2 passphrase over the encrypted channel
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

### WireGuard VPN — wg-SE-RO-1

- **Interface:** `wg-SE-RO-1` (replaces legacy `wg0`)
- **Management:** NetworkManager — the tunnel is treated as a first-class network connection,
  enabling persistent roaming and DNS integration
- **Routing:** All external traffic routed through the tunnel; kill-switch rule drops traffic
  if the interface drops

The rename from `wg0` to `wg-SE-RO-1` reflects a move to a named-tunnel model where the
identifier encodes the endpoint region, making multi-tunnel configurations readable.

### AdGuard Home DNS Filtering

- **Upstream DNS:** WireGuard peer at `10.2.0.1` — DNS queries traverse the encrypted tunnel
  before reaching a resolver
- **Listener:** `127.0.0.1:53` — no DNS traffic leaves the host unfiltered
- **Block lists:** Standard tracking, malware, and telemetry lists enabled
- **systemd-resolved:** Configured to forward all queries to `127.0.0.1` — AdGuard Home
  intercepts before anything reaches the upstream

The DNS chain: application → systemd-resolved (loopback) → AdGuard Home → WireGuard →
upstream resolver. An attacker observing the external interface sees only WireGuard-encrypted
traffic; the upstream resolver sees the WireGuard exit IP, not the host IP.

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
<!-- MANUAL INPUT REQUIRED: record the udev rule enabling non-root OpenRGB HID access (OpenRGB ships 60-openrgb.rules; confirm the installed path under /usr/lib/udev/rules.d/) -->

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
| Disk | LUKS2 volume on `/dev/sda3` with Argon2id KDF | Offline disk clone and brute-force key derivation |
| Boot | dracut-sshd pre-boot SSH with pinned ed25519 host key | Unattended unlock without exposing the passphrase; MITM key substitution during unlock |
| Key custody | NK#1 primary slot, NK#2 offline backup slot, offline emergency passphrase | Single point of key failure; primary key loss or seizure |
| Network | WireGuard `wg-SE-RO-1` (NetworkManager-managed) with kill-switch | Traffic interception, geolocation, and leakage if the tunnel drops |
| DNS | AdGuard Home on `127.0.0.1:53` + systemd-resolved loopback binding; upstream via `10.2.0.1` over WireGuard | Plaintext DNS leakage; tracker, telemetry, and known-malicious domain resolution |

The hardware key is the root of the trust chain. Without a Nitrokey or the emergency
passphrase, the disk does not open. The emergency passphrase is stored offline and is
not present on the machine in any form.

<!-- FUTURE WORK: document TPM integration if Measured Boot is added in a future iteration (current build does not bind unlock to PCR measurements — see hardening/research/luks2-fido2-reference.md §5) -->
<!-- MANUAL INPUT REQUIRED: record the SELinux/seccomp policy status — run `getenforce` and `sestatus` (Fedora ships SELinux enforcing by default; confirm) -->

---

## Results and Current State

| Component | Status |
|-----------|--------|
| LUKS2 with Argon2id | Operational — all three keyslots active |
| Nitrokey NK#1 FIDO2 enrollment | Operational — firmware 1.8.3, touch-only |
| Nitrokey NK#2 backup keyslot | Enrolled — stored offline |
| dracut-sshd remote unlock | Operational — Termux ed25519 key in initramfs |
| WireGuard wg-SE-RO-1 | Operational — NetworkManager-managed |
| AdGuard Home | Operational — upstream via 10.2.0.1 |
| systemd-resolved → 127.0.0.1 | Operational |
| OpenRGB (Razer + Corsair) | Operational — vendor daemons absent |

<!-- MANUAL INPUT REQUIRED: add LUKS2 unlock-latency benchmark data (hardware key vs passphrase) measured on the running machine -->
<!-- MANUAL INPUT REQUIRED: record the Fedora release and kernel — `cat /etc/fedora-release` and `uname -r` -->
<!-- MANUAL INPUT REQUIRED: record the dracut-sshd build command and initramfs rebuild trigger — see hardening/dracut-sshd.md (`sudo dracut -f --regenerate-all`) and confirm the config drop-in -->

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
