# LUKS2 Full-Disk Encryption Setup

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running machine; `FUTURE WORK` markers flag planned enhancements.

## Encrypted volume

| Attribute | Value |
|-----------|-------|
| Device | `/dev/sda3` |
| Format | LUKS2 |
| Key derivation function | Argon2id |
| Cipher | aes-xts-plain64 (LUKS2 default) <!-- MANUAL INPUT REQUIRED: confirm exact cipher and key size — run `sudo cryptsetup luksDump /dev/sda3` and read Data segments → cipher (expect aes-xts-plain64) and key size (expect 512-bit) --> |

Argon2id is memory-hard, making offline brute-force of a cloned disk impractical without the
hardware key or the emergency passphrase.

## Keyslot layout

Three keyslots provide redundancy without weakening the security model:

| Slot | Type | Role |
|------|------|------|
| 0 | Passphrase | Emergency fallback only — long, high-entropy, stored offline |
| 1 | Nitrokey NK#1 (primary) | Daily driver — physical touch required for every unlock |
| 2 | Nitrokey NK#2 (backup) | Kept offline; activated only if NK#1 is lost |

Inspect the header with:

```
sudo cryptsetup luksDump /dev/sda3
```

## FIDO2 enrollment

The hardware keyslots are enrolled with `systemd-cryptenroll` using FIDO2. The LUKS2 passphrase
is never transmitted to the key; the Nitrokey produces a FIDO2 credential combined with a stored
key file to derive the slot key.

```
# Primary hardware key (NK#1)
sudo systemd-cryptenroll /dev/sda3 --fido2-device=auto

# Backup hardware key (NK#2) — enrolled while NK#2 is inserted
sudo systemd-cryptenroll /dev/sda3 --fido2-device=auto
```

**Touch-only enrollment:** the Nitrokey 3A NFC on firmware 1.8.3 does **not** support
`clientPin`. Enrollment therefore relies on physical touch confirmation rather than a
PIN-gated credential. This is intentional — a PIN-over-USB credential could be activated
without physical presence, defeating the presence guarantee. See
[nitrokey.md](nitrokey.md) for the hardware-key detail.

```
# clientPin is not enrolled; presence is enforced by touch
sudo systemd-cryptenroll /dev/sda3 --fido2-device=auto --fido2-with-client-pin=no
```

## crypttab configuration

`/etc/crypttab` references the volume and instructs systemd to use the FIDO2 device, falling
back to the passphrase prompt if no key is present:

```
# <name>   <device>     <keyfile>  <options>
luks-sda3  /dev/sda3    none       fido2-device=auto,luks,discard
```

<!-- MANUAL INPUT REQUIRED: confirm the final crypttab options (timeout, tries, and whether discard is enabled given the SSD/threat trade-off) — copy the live line from /etc/crypttab -->

## Verification

```
sudo cryptsetup luksDump /dev/sda3          # confirm 3 active keyslots
sudo systemd-cryptenroll /dev/sda3 --fido2-device=list
lsblk -f                                     # confirm the mapped volume
```

## Current state

| Item | Status |
|------|--------|
| LUKS2 + Argon2id on `/dev/sda3` | Operational |
| Slot 0 — emergency passphrase | Active; stored offline |
| Slot 1 — NK#1 FIDO2 (touch-only) | Operational |
| Slot 2 — NK#2 FIDO2 backup | Enrolled; stored offline |

<!-- MANUAL INPUT REQUIRED: benchmark unlock latency (hardware key vs passphrase) — time each unlock path on the running machine and record the figures -->
<!-- MANUAL INPUT REQUIRED: record the Fedora release and kernel this was built on — run `cat /etc/fedora-release` and `uname -r` -->
