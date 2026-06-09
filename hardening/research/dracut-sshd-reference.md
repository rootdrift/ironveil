# dracut-sshd — Remote Initramfs Unlock Architecture Reference

Technical reference for the remote-unlock path described in
[../dracut-sshd.md](../dracut-sshd.md). Explains how an SSH-accessible initramfs works, its threat
surface, and how it compares to other remote/automated unlock methods. Reference document, not a
build log.

---

## 1. The boot-time problem this solves

On a full-disk-encrypted machine the root filesystem is locked until the LUKS2 volume is unlocked.
That normally requires someone at the physical console to type a passphrase or touch a hardware
key. For a headless or remotely-administered workstation that means it cannot reboot
unattended — every reboot blocks at the passphrase prompt.

The initramfs is the small, **unencrypted** root environment the kernel mounts first. It already
contains the logic to prompt for and apply the LUKS passphrase. `dracut-sshd` extends that
environment with a network stack and an SSH server, so the unlock prompt can be answered *over the
network* instead of at the keyboard.

IRONVEIL uses this to unlock the workstation from the GrapheneOS handset (Termux) — see the
cross-build note in [../../README.md](../../README.md) and nullbyte's Termux/SSH integration.

---

## 2. Architecture

### 2.1 What dracut bakes into the initramfs

`dracut` builds the initramfs as a cpio archive. The `dracut-sshd` module adds, at build time:

1. **An SSH daemon** (the module ships an OpenSSH `sshd` build suited to the minimal environment)
   configured to start once the network is up and before the root-mount/unlock step blocks.
2. **A host key** for that initramfs sshd. This is generated/placed at build time and is **baked
   into the image**. Because it lives in the unencrypted initramfs, it is readable by anyone who
   can read the boot partition (see §3).
3. **An `authorized_keys` file** containing the *public* key(s) permitted to connect. IRONVEIL
   bakes in the public half of a dedicated **ed25519** key generated in Termux on the GrapheneOS
   device.
4. **Network configuration** so the interface comes up early. This is driven by the kernel command
   line `ip=` directive and the appropriate network dracut module (e.g. `network-manager` or the
   legacy `network`/`network-legacy` module). Static or DHCP addressing is set here.

### 2.2 Boot sequence with dracut-sshd

```
Power on
  └─ Firmware → bootloader (GRUB/sd-boot) → kernel + initramfs (all UNENCRYPTED)
       └─ initramfs starts
            ├─ udev brings up the NIC; ip= applies address/route
            ├─ dracut-sshd starts sshd (listening, pre-unlock)
            └─ systemd-cryptsetup blocks, waiting for the LUKS2 key
                 ▲
                 │  (admin connects from GrapheneOS Termux over SSH)
                 │  ssh unlock@$IRONVEIL_IP
                 │  → systemd-tty-ask-password-agent  (supplies passphrase)
                 ▼
            LUKS2 volume unlocks → real root pivots in → sshd in initramfs is torn down
                 └─ normal multi-user boot continues
```

### 2.3 Why the network must come up *before* root mount

The unlock has to happen while `systemd-cryptsetup` is still blocking. If networking initialised
only in the real root, you would already need the disk unlocked to get a network — a chicken/egg
problem. dracut-sshd therefore depends on early-boot networking inside the initramfs, configured
via `ip=` on the kernel cmdline and the network dracut module. *Verify in this build which network
module and `ip=` form are used (DHCP vs static), and record it in the build notes.*

---

## 3. Threat surface of an SSH-accessible initramfs

An internet- or LAN-reachable SSH service that exists *before the disk is unlocked* is a real
attack surface and must be reasoned about honestly.

### 3.1 What an attacker with network access **can** see / attempt

- **Read the initramfs offline.** The initramfs (and its baked-in sshd host key and
  `authorized_keys`) live unencrypted on the boot partition. An attacker who can read that
  partition (e.g. stolen drive, or a separate unencrypted `/boot`) obtains the **host private key**
  and the **authorized public key**. The host private key matters for impersonation (see §3.3); the
  authorized *public* key does not help them authenticate (it is public).
- **Reach the listening port** during the unlock window and attempt authentication.
- **Probe the sshd version** and any banner.

### 3.2 What the attacker **cannot** do

- **Authenticate without the client private key.** Auth is public-key only against the baked-in
  ed25519 public key. Password auth should be disabled (verify in this build). Without the Termux
  private key, no session.
- **Read disk contents.** Pre-unlock there is nothing decrypted to read; the data volume's MK is
  not yet in memory.
- **Brute-force the LUKS key over SSH.** Even with a session, the unlock is via
  `systemd-tty-ask-password-agent`, which feeds the passphrase prompt — it does not expose the MK.

### 3.3 The real risks to manage

1. **Host-key theft → MITM.** Because the host key is baked into a readable initramfs, an attacker
   who has read `/boot` could stand up a fake "unlock" SSH endpoint presenting the *same* host key
   and, if they can also intercept the admin's connection (e.g. on a hostile network), capture the
   **LUKS passphrase** when the admin types it into the spoofed session. **Mitigation, and the one
   IRONVEIL uses:** pin the initramfs host key on the client so a substitution is detected — but
   note that pinning protects against a *different* host key, not against reuse of the *stolen
   genuine* key. The stronger mitigation is to treat the initramfs host key as low-trust and never
   send a reusable secret through it; the FIDO2-derived value or a dedicated unlock token is
   preferable to typing the master passphrase. *Open item: confirm whether the remote path supplies
   the emergency passphrase (reusable, higher value if captured) or a dedicated/throwaway unlock
   secret.*
2. **Exposure window.** The sshd is reachable to anyone who can route to the host during every
   boot. Restrict reachability: bind to the management interface/VPN only, firewall the initramfs
   listener to known source addresses, or only expose the unlock service across the WireGuard path
   rather than the open LAN/Internet.
3. **Stale baked-in keys.** The authorized key and host key are fixed until the next
   `dracut -f`. Rotation is a manual, deliberate step (§5).

---

## 4. Comparison with other remote / automated unlock methods

| Method | How it unlocks | Interaction | Strengths | Weaknesses / fit |
|--------|----------------|-------------|-----------|------------------|
| **dracut-sshd** (this build) | Human connects via SSH to the initramfs and answers the unlock prompt | Manual, per boot | Full control; no on-network secret store; uses existing SSH/Termux workflow | SSH attack surface pre-unlock; host key in readable initramfs; manual every boot |
| **dropbear-initramfs** (Debian/Ubuntu) | Same idea, but a tiny Dropbear SSH server instead of OpenSSH | Manual, per boot | Very small footprint; long-established on Debian | Same exposure model; Dropbear feature/crypto set is smaller than OpenSSH |
| **clevis-tang (NBDE)** | Initramfs reconstructs the key via a McCallum–Relyea exchange with a network *tang* server | **None** (automatic on the trusted network) | Unattended reboots; no stored key; unlocks only on the trusted network | Requires a tang server; unlocks automatically for anyone who boots on that network — no presence factor |
| **clevis-tpm2 / systemd-tpm2** | Key sealed to the local TPM, released if PCR measurements match | **None** (automatic) | Transparent; binds to boot integrity (tamper breaks unlock) | Auto-unlocks for whole-machine thief unless a PIN/PCR policy is added; TPM bus-sniffing risk |
| **clevis sss (Shamir threshold)** | Threshold combination of TPM + tang + passphrase | Configurable | Policy flexibility (e.g. "TPM **and** tang") | More moving parts to get right |

**Where dracut-sshd fits:** it is the right tool when you want a **human-in-the-loop** remote
unlock with no automatic key release and no dependence on a tang/TPM infrastructure — i.e. a
roaming workstation you administer from a phone. Where you want *unattended* reboots, tang or TPM
sealing are the appropriate tools, at the cost of removing the presence factor.

dropbear-initramfs is the closest equivalent; the choice of OpenSSH-based dracut-sshd on Fedora is
mostly an ecosystem/feature-parity decision (Fedora uses dracut; the module integrates cleanly with
systemd in the initramfs).

---

## 5. `systemd-tty-ask-password-agent` — why it is the correct unlock command

When `systemd-cryptsetup` needs a key it does not read the terminal directly. It publishes a
**password request** through systemd's password-agent mechanism (request files under
`/run/systemd/ask-password/`, watched via inotify). Any registered agent can answer it.

`systemd-tty-ask-password-agent` is the agent that:

- enumerates the outstanding password requests, and
- with `--query` prompts for and delivers the answer to the waiting unit over the proper systemd
  channel (not by poking at a TTY device the SSH session does not own).

Why it is correct over the alternatives:

- **`echo … > /dev/console` or writing to a TTY** is fragile: the cryptsetup prompt is owned by the
  password-agent subsystem, not a plain TTY the SSH session can write to. It may not be received,
  and races with the agent.
- **Restarting the cryptsetup unit / re-running cryptsetup by hand** bypasses the orchestrated boot
  and can leave systemd's dependency state inconsistent.
- `systemd-tty-ask-password-agent --query` (or simply running the agent) speaks the supported
  protocol, so the answer is routed to exactly the unit that is blocking, and boot continues
  cleanly.

Practical note: some setups invoke it as `systemd-tty-ask-password-agent --query --watch` so the
agent stays attached for the duration; confirm the exact invocation used and document it.

---

## 6. Operational security

### 6.1 Key rotation

- **Client (Termux) key:** the build rotates the ed25519 unlock key whenever the initramfs is
  rebuilt. Rotation procedure: generate a new keypair in Termux → replace the public key in the
  dracut config / `authorized_keys` source → `sudo dracut -f --regenerate-all` → re-pin the new
  host key on the client → securely destroy the old private key. Track this as a checklist item so
  a rebuild never silently re-bakes a stale or compromised key.
- **Initramfs host key:** ideally regenerated on rebuild as well, since it is the impersonation
  target (§3.3). If the host key is regenerated, the client's pinned known-hosts entry must be
  updated in the same operation.

### 6.2 If the unlock (client) key is compromised

Compromise of the *client private key* means an attacker who can also reach the initramfs sshd
could open a pre-unlock session. They still cannot unlock the disk without the LUKS secret, but
they gain a foothold to attempt the MITM/passphrase-capture path. Response:

1. Rebuild the initramfs with a fresh `authorized_keys` (revokes the old client key) — `dracut -f`.
2. Rotate the initramfs host key in the same rebuild and re-pin on the client.
3. Treat the LUKS **passphrase** (slot 0) as potentially exposed if it was ever typed through a
   session that the attacker could have MITM'd, and change it (`cryptsetup luksChangeKey`).
4. Constrain network reachability of the initramfs sshd (VPN-only / source-IP firewalling) to
   shrink the window.

### 6.3 Reducing exposure generally

- Expose the initramfs listener only over the trusted/management path (e.g. via WireGuard or a
  restricted source-address set) rather than the open network.
- Disable password authentication entirely in the initramfs sshd (public-key only).
- Keep the unlock secret delivered over SSH as low-value as possible relative to the master
  passphrase (see §3.3 open item).
- Pair with Secure Boot / measured boot to address the evil-maid risk that an SSH-in-initramfs
  design does not, by itself, cover — the initramfs is unencrypted and tamperable
  (cross-reference [luks2-fido2-reference.md](luks2-fido2-reference.md) §3.2).

---

## 7. Open items / to verify

- [ ] Which network dracut module + `ip=` form is used (DHCP vs static), and on which interface.
- [ ] Whether the remote path delivers the reusable emergency passphrase or a dedicated unlock
      secret (informs the MITM blast radius in §3.3).
- [ ] Exact `systemd-tty-ask-password-agent` invocation in use (`--query`, `--watch`).
- [ ] Whether the initramfs sshd is reachable only over WireGuard / restricted source IPs.
- [ ] Whether the initramfs host key is rotated on each rebuild alongside the client key.
- [ ] Whether password auth is disabled in the initramfs sshd.

> No package versions, CVEs, or vendor claims are asserted here without verification; the package
> source/version for `dracut-sshd` itself remains a TODO in the build notes.
