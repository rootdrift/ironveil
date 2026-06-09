# dracut-sshd — Remote LUKS Unlock over SSH

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running machine; `FUTURE WORK` markers flag planned enhancements.

## Purpose

`dracut-sshd` starts a minimal SSH daemon inside the initramfs, *before* the LUKS2 volume is
mounted. This allows the encrypted workstation to be unlocked remotely from the GrapheneOS
handset (Termux) without ever transmitting the passphrase in cleartext and without physical
presence at the machine. Tested working.

## Installation

`dracut-sshd` is added to the dracut module set so the initramfs includes a pre-boot SSH
listener:

```
# Install the dracut-sshd module (Fedora / COPR)
# MANUAL INPUT REQUIRED: pin the package source (COPR repo) and version used in this build
#   — run `rpm -q dracut-sshd` and record the COPR repo (e.g. `dnf copr list --enabled`)
sudo dnf install dracut-sshd

# Rebuild the initramfs to embed the SSH daemon and authorized key
sudo dracut -f --regenerate-all
```

<!-- MANUAL INPUT REQUIRED: document the dracut config drop-in and network setup — `cat /etc/dracut.conf.d/*.conf` (the sshd/network module config) and `cat /proc/cmdline` (the `ip=` directive and network module on the running kernel) -->

## Key material

- A dedicated **ed25519** key pair is generated in Termux on the GrapheneOS device.
- The **public** key is baked into the initramfs `authorized_keys` at build time.
- This key is **separate** from all other SSH keys on the system and is rotated whenever the
  initramfs is rebuilt.
- The initramfs SSH **host key** is pinned on the GrapheneOS client to prevent MITM substitution
  during the unlock exchange.

## Unlock flow

1. The workstation powers on and reaches the initramfs SSH listener (pre-mount).
2. From GrapheneOS Termux: `ssh unlock@$IRONVEIL_IP`.
3. The connection authenticates against the embedded ed25519 public key.
4. Inside the session, the password agent is signalled to supply the LUKS2 passphrase over the
   encrypted channel:

   ```
   systemd-tty-ask-password-agent
   ```

5. The LUKS2 volume unlocks and boot continues into userspace.

## Security properties

| Property | Mechanism |
|----------|-----------|
| Passphrase never sent in cleartext | Supplied only inside the encrypted SSH session |
| Two-factor unlock path | Physical GrapheneOS device (have) + Termux key file (have, separate) |
| MITM resistance | Initramfs SSH host key pinned on the client |
| Key isolation | Initramfs ed25519 key distinct from all other SSH keys; rotated on rebuild |

An attacker with network access but without the GrapheneOS device and its Termux key cannot
complete the unlock.

## Current state

| Item | Status |
|------|--------|
| dracut-sshd in initramfs | Operational |
| Termux ed25519 key embedded | Operational |
| Remote unlock via `systemd-tty-ask-password-agent` | Tested working |
| Host-key pinning on client | Operational |

<!-- MANUAL INPUT REQUIRED: document the initramfs rebuild trigger and the ed25519 key-rotation procedure (generate new key in Termux → update authorized_keys → `sudo dracut -f --regenerate-all` → re-pin host key on the client); cross-reference the nullbyte Termux SSH integration -->
