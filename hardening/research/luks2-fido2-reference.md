# LUKS2 + FIDO2 — Cryptographic Architecture Reference

Technical reference informing the IRONVEIL disk-encryption build. This document explains the
mechanisms behind the configuration in [../luks2-setup.md](../luks2-setup.md) and
[../nitrokey.md](../nitrokey.md). It is a research/reference document, not a build log — where a
value is implementation-specific it is flagged for verification against the live header
(`cryptsetup luksDump /dev/sda3`).

---

## 1. LUKS2 cryptographic architecture

### 1.1 The master-key / keyslot separation

LUKS (Linux Unified Key Setup) does **not** encrypt the disk directly with a passphrase or a
hardware-key output. It uses a two-tier model:

- **Volume key (master key, MK):** a single high-entropy random key generated at format time.
  This is the key that actually encrypts the data via dm-crypt. For `aes-xts-plain64` the MK is
  512 bits (XTS uses two 256-bit AES keys).
- **Keyslots:** each keyslot stores an *encrypted copy* of the master key. The encryption key for
  a keyslot — the "slot key" — is derived from a passphrase (via a PBKDF) or supplied by a token
  (FIDO2, TPM, etc.).

Consequences of this design:

- Multiple keyslots can unlock the *same* volume independently. IRONVEIL uses three (passphrase,
  NK#1, NK#2) — all decrypt the same MK.
- Changing a passphrase re-encrypts only that keyslot's copy of the MK; it does **not** re-encrypt
  the disk.
- Revoking a key means wiping its keyslot. The data is not re-keyed, so revocation assumes the
  old slot key was not captured *and* the MK was not extracted while the volume was open.

### 1.2 LUKS2 on-disk structure

LUKS2 replaced the fixed binary header of LUKS1 with a JSON metadata model:

| Region | Contents |
|--------|----------|
| Binary header (×2) | Magic, version, header size, MK digest reference, offset pointers. Stored in two copies (primary + secondary) for resilience against partial corruption. |
| JSON metadata area | Describes keyslots, segments (the encrypted data area + cipher), digests, and **tokens** (e.g. the FIDO2 credential metadata). Also stored in two copies. |
| Keyslots area | The anti-forensically split, encrypted copies of the master key. |

LUKS2 supports up to 32 keyslots (LUKS1 supported 8). The JSON token objects are how FIDO2/TPM
enrolments attach non-passphrase unlock data to the header — see §2.4.

### 1.3 Anti-forensic (AF) splitter

Before a keyslot's MK copy is written, it is processed by the **anti-forensic information
splitter** (Clemens Fruhwirth's design). The key material is expanded (default 4000 stripes) and
diffused so that recovering the key requires *every* stripe intact. The purpose is to defeat
recovery of a keyslot from a single overwritten sector or from worn flash blocks — partial
recovery yields nothing. This is why a `cryptsetup luksKillSlot` that overwrites the keyslot area
is meaningful even on media with imperfect overwrite guarantees, though on SSDs wear-levelling
means the only hard guarantee is destroying the MK / header.

### 1.4 Key derivation function — Argon2id

LUKS2's default PBKDF is **Argon2id**, the hybrid variant of Argon2 (Password Hashing Competition
winner, RFC 9106). Argon2id combines:

- **Argon2i** resistance to side-channel (data-independent memory access in the first pass), and
- **Argon2d** resistance to GPU/ASIC time-memory trade-off attacks (data-dependent access later).

It is **memory-hard**: an attacker must commit a large amount of RAM *per guess*, which is what
makes massively-parallel offline cracking of a cloned disk economically impractical compared to
the older PBKDF2 (LUKS1 default), which is only CPU-bound.

Three cost parameters:

| Parameter | Meaning | cryptsetup default behaviour |
|-----------|---------|------------------------------|
| Time cost (iterations) | Number of passes over memory | Auto-tuned so one derivation takes ~2000 ms on the formatting machine (`--iter-time 2000`) |
| Memory cost | KiB of RAM per derivation | Auto-benchmarked, capped at a default maximum (commonly up to ~1 GiB / 1048576 KiB) |
| Parallelism | Lanes / threads | Typically min(4, available CPU threads) |

> **Verify against the live header.** The exact Argon2id time/memory/parallelism values are
> chosen at format time by `cryptsetup`'s runtime benchmark and depend on the machine. Read the
> actual values with `cryptsetup luksDump /dev/sda3` (the per-keyslot `kdf:` block shows
> `type: argon2id`, `time:`, `memory:`, `cpus:`, and `salt:`). These should be recorded in
> [../luks2-setup.md](../luks2-setup.md) rather than assumed.

Important nuance: the Argon2id cost only protects **passphrase** keyslots, because only those
derive the slot key from a low-entropy human secret. A FIDO2 keyslot's input is already a
high-entropy 256-bit value from the authenticator (see §2), so its KDF hardening matters far less
— brute-forcing a 256-bit space is infeasible regardless of KDF.

---

## 2. FIDO2 as a LUKS2 unlock factor

### 2.1 The mechanism: `hmac-secret`, not credential storage

The single most important technical point — and a refinement of the looser "credential combined
with a key file" phrasing in the build notes — is that `systemd-cryptenroll` uses the FIDO2/CTAP2
**`hmac-secret` extension**. The flow:

1. At enrolment, `systemd-cryptenroll` creates a FIDO2 credential on the authenticator and stores
   in the LUKS2 **token** metadata: the *credential ID*, a random *salt*, the relying-party ID,
   and which factors are required (user presence / PIN / user verification).
2. At unlock, systemd sends the credential ID + salt to the authenticator.
3. The authenticator computes `HMAC-SHA256(per-credential-secret, salt)` **inside the device**.
   The per-credential secret never leaves the hardware.
4. The 32-byte HMAC output is used by systemd as the high-entropy key material to unlock that
   LUKS2 keyslot (after systemd's own hashing/handling).

So the secret that actually unlocks the slot is *derived on the token from a salt stored in the
header*. There is no exportable key file; possession of the disk alone yields only the salt and
credential ID, which are useless without the device that holds the per-credential secret.

### 2.2 Resident vs non-resident credentials

- **Non-resident (default for `systemd-cryptenroll`):** the credential ID is stored in the LUKS2
  token. The authenticator does not need to store anything per-credential beyond its master seed;
  it can re-derive the credential from the credential ID + its internal key. This means the
  enrolment consumes none of the device's limited resident-credential (passkey) slots.
- **Resident / discoverable:** the credential is stored on the device itself. Not required here
  and would consume a passkey slot.

IRONVEIL's enrolment is the standard non-resident path — the credential IDs live in the LUKS2
header, one per enrolled key (NK#1, NK#2).

### 2.3 The libfido2 / CTAP2 stack

```
systemd-cryptsetup / systemd-cryptenroll
        │  (links against)
     libfido2  ──────────────►  fido2-token / fido2-assert / fido2-cred (CLI equivalents)
        │
     CTAP2 (Client to Authenticator Protocol)
        │  transport: USB-HID (or NFC/BLE)
        ▼
   Nitrokey 3A NFC authenticator  ── computes hmac-secret internally
```

- **CTAP2** is the protocol spoken between host and authenticator; `hmac-secret` is a CTAP2
  extension.
- **libfido2** (Yubico) is the host-side library implementing CTAP. systemd depends on it for the
  FIDO2 enrol/unlock path.
- FIDO2 = WebAuthn (browser/relying-party side) + CTAP (authenticator side). For disk encryption
  only the CTAP/`hmac-secret` half is used; there is no web relying party.

### 2.4 The LUKS2 token object

The enrolment writes a `systemd-fido2` token into the header JSON. It records the credential ID,
salt, RP ID, and the required user factors (`fido2-clientPin-required`,
`fido2-up-required` for user presence, `fido2-uv-required` for user verification). At boot,
`systemd-cryptsetup` reads this token, finds a matching FIDO2 device, and runs the assertion. This
is why no entry beyond `fido2-device=auto` is needed in `crypttab` — the parameters live in the
header token, not the config file.

---

## 3. Security analysis

### 3.1 What FIDO2-backed LUKS protects against (vs passphrase alone)

| Threat | Passphrase alone | FIDO2 keyslot |
|--------|------------------|---------------|
| Offline brute force of a cloned/stolen disk | Depends entirely on passphrase entropy + Argon2id cost | Infeasible — slot key is a 256-bit device-derived value |
| Keyloggers / shoulder-surfing / camera capture at unlock | Captures the passphrase | Nothing typed; capture is meaningless without the device |
| Passphrase reuse / phishing of the passphrase | Real risk | No transferable secret to phish |
| Weak human-chosen passphrase | Primary weakness | Eliminated for the hardware slots |

### 3.2 What it does **not** protect against

This is the honest threat-model boundary and must be stated plainly:

- **Evil-maid / boot-chain tampering.** LUKS encrypts data at rest, not the boot chain. An
  attacker with transient physical access can modify the unencrypted bootloader/initramfs (and on
  IRONVEIL the initramfs also contains the dracut-sshd SSH server — see
  [dracut-sshd-reference.md](dracut-sshd-reference.md)) to capture the unlock or the MK on next
  use. Mitigation is Secure Boot + signed/measured boot, **not** LUKS itself. *Status to verify in
  this build: whether Secure Boot and a measured/signed initramfs are in place.*
- **Cold-boot / DMA attacks while running.** Once unlocked, the MK is in RAM. Cold-boot RAM
  remanence or a DMA-capable port (Thunderbolt/PCIe) can extract it. Mitigations: IOMMU/VT-d,
  disabling unused DMA ports, and locking/powering off rather than suspending.
- **Coercion ("$5 wrench").** A user who can be compelled to touch the key and boot the machine
  defeats any at-rest scheme.
- **Theft of the key *together with* the machine.** This is the critical consequence of
  **touch-only** enrolment (no clientPin — see §4). The FIDO2 slot requires only *user presence*
  (a touch), not *user verification* (a PIN). Anyone in physical possession of the Nitrokey **and**
  the workstation can unlock the disk by touching the key. FIDO2 here proves "have the device,"
  not "know a secret." The emergency passphrase (slot 0) remains the only knowledge factor.
- **Attacks against an unlocked, running system.** Malware, privilege escalation, and live
  forensics all operate after unlock and are out of scope for FDE.

### 3.3 Single-factor vs two-factor framing

With touch-only FIDO2, the hardware keyslot is effectively **single-factor: something you have**.
To make disk unlock genuinely two-factor (have + know) you would enrol with a clientPin
(`--fido2-with-client-pin=yes`), which the current firmware path does not use (§4). IRONVEIL's
compensating control is operational: physical separation of the backup key (NK#2 in the Pelican
1200 case) and custody discipline for NK#1, plus the dracut-sshd remote-unlock path which adds a
*separate* device + key as a second possession factor for the network unlock route.

---

## 4. Nitrokey 3A NFC, firmware 1.8.3

| Property | Detail |
|----------|--------|
| Platform | Nitrokey 3 family, open-source firmware (Trussed framework, Rust) |
| Secure element | NXP secure element + LPC55 microcontroller (the 3A NFC pairs an SE with the MCU) |
| FIDO2 | CTAP2 with `hmac-secret` extension — the capability LUKS unlock relies on |
| NFC | Present (the "NFC" SKU); USB-A form factor for the 3A |

### 4.1 clientPin absence on this firmware path and its implications

The build notes record that the 1.8.3 firmware path is used **touch-only**, with `clientPin` not
enrolled, and `systemd-cryptenroll … --fido2-with-client-pin=no`.

Accurate framing of what this does and does not mean:

- **What touch (user presence) gives you:** it prevents *silent/remote/automated* activation. An
  assertion cannot be completed by software alone over a connected USB bus; a human must physically
  touch the key. This is what blocks malware on a *running, already-unlocked* host from quietly
  exercising the credential.
- **What a clientPin (user verification) would add:** a second factor (*something you know*) so
  that physical possession of the key is insufficient — a thief with the key still could not
  unlock without the PIN. Its absence is exactly why §3.2's "key + machine stolen together"
  scenario succeeds.
- **On the stated rationale.** The build note frames touch-only as preventing remote activation.
  That outcome is real, but technically it is *user presence* (touch), not the *absence of a PIN*,
  that delivers it — a PIN and user presence are independent factors and enrolling a PIN would not
  have weakened presence. The accurate statement is: "presence is enforced by touch; a PIN was not
  added, so the hardware slots are single-factor possession." **This reasoning should be
  reviewed/confirmed** rather than treated as settled, including whether the 1.8.3 firmware in use
  fully supported `hmac-secret` together with clientPin at build time (early Nitrokey 3 firmware
  had evolving FIDO2 feature coverage — verify before relying on the rationale).

### 4.2 Touch-only enrolment rationale (as a deliberate trade-off)

Choosing presence-only is defensible **if** the operational model is "the workstation and the key
are not in the same place when unattended, and the backup key + passphrase live in separate
physical custody." That matches the documented NK#1-carried / NK#2-in-Pelican-1200 / offline
passphrase arrangement. The residual risk (combined theft) is accepted in exchange for fast,
PIN-free daily unlock. Documenting it as an accepted risk — rather than implying touch-only is
strictly stronger — is the correct posture.

---

## 5. FIDO2 vs TPM-based disk encryption

Both bind disk unlock to hardware; they answer different threat models.

| Dimension | FIDO2 (`systemd-cryptenroll --fido2`) | TPM (clevis-tpm2, systemd-tpm2, BitLocker) |
|-----------|----------------------------------------|---------------------------------------------|
| Secret location | External, removable authenticator | Soldered TPM on the mainboard |
| User interaction | Required (touch); optional PIN | **None by default** — auto-unlock at boot |
| Binds to boot integrity? | No (proves possession of a token) | Yes — key sealed to PCR measurements of firmware/bootloader/kernel; tamper breaks unlock |
| Disk theft (drive only) | Safe — secret not on the drive | Safe — key not on the drive, sealed in TPM |
| Whole-machine theft | TPM auto-unlocks unless PIN/PCR gate set; FIDO2 needs the external key (and touch) | TPM-only configs unlock for the thief who powers it on |
| Notable attack | Combined key+machine theft (touch-only) | Bus-sniffing the TPM (LPC/SPI) to capture the key in transit on TPM-only setups; PCR rollback |
| Best fit | Laptops/workstations where a human is present to unlock and theft of the portable key is controlled | Unattended servers, fleets, and "transparent" UX where measured boot is the goal |

Other NBDE options:

- **clevis-tang (Network-Bound Disk Encryption):** the key is reconstructed via a McCallum-Relyea
  exchange with a network "tang" server; the disk unlocks **only on the trusted network**. Ideal
  for datacenter fleets that must reboot unattended but should not unlock if a drive leaves the
  building. Not appropriate for a roaming laptop.
- **clevis with `sss` (Shamir):** combine TPM + tang + passphrase with a threshold policy.

**Why FIDO2 for IRONVEIL:** it is a portable workstation where (a) a human is present to unlock,
(b) auto-unlock is undesirable (a stolen powered-off laptop must not boot to a usable state), and
(c) the build deliberately avoids trusting a fixed on-board secret. The trade-off accepted is the
lack of boot-integrity binding (no PCR sealing) — which is why Secure Boot / measured boot is the
natural next hardening step and should be tracked as such.

---

## 6. Verification commands

```bash
# Inspect KDF parameters, keyslots, and the FIDO2 token object
sudo cryptsetup luksDump /dev/sda3

# List FIDO2 enrolments recognised on the header
sudo systemd-cryptenroll /dev/sda3 --fido2-device=list

# Inspect the connected authenticator's CTAP capabilities (confirm hmac-secret support)
fido2-token -I /dev/hidrawX     # look for the 'hmac-secret' extension in 'extensions'
nitropy fido2 list
```

---

## 7. Open items / to verify

- [ ] Record actual Argon2id `time` / `memory` / `cpus` from `luksDump` into the build notes.
- [ ] Confirm exact cipher and key size (`aes-xts-plain64`, 512-bit) from `luksDump` segments.
- [ ] Confirm whether the 1.8.3 firmware supported `hmac-secret` + clientPin together at build
      time; revisit the touch-only rationale accordingly (§4.1).
- [ ] Decide whether to add Secure Boot + measured/signed initramfs to close the evil-maid gap.
- [ ] Document combined-theft as a formally accepted residual risk in the threat model.

> Nothing in this document asserts a CVE, a specific cracking benchmark, or a vendor claim that
> has not been verified. Items that depend on the live configuration or on firmware behaviour are
> flagged above rather than stated as fact.
