# dracut-sshd — Remote LUKS Unlock over SSH

Living build — hardware values captured from the running machine on 2026-06-11. `FUTURE WORK`
markers flag planned enhancements.

## Purpose

`dracut-sshd` starts a minimal SSH daemon inside the initramfs, *before* the LUKS2 volume is
mounted. This allows the encrypted workstation to be unlocked remotely — the GrapheneOS handset
(Termux) reaches the pre-boot listener over the network and triggers the unlock, without ever
transmitting the passphrase in cleartext and without standing at the machine. The initramfs also
carries the `fido2` module, so a Nitrokey touch can unlock directly at the prompt. Tested working.

## Installation

Installed version: **`dracut-sshd-0.7.1-5.fc44.noarch`**, from the **official Fedora repository**
— *not* the upstream COPR. Confirmed 2026-06-12:

```
$ rpm -qi dracut-sshd | grep -E 'Vendor|Packager|Source RPM'
Source RPM  : dracut-sshd-0.7.1-5.fc44.src.rpm
Packager    : Fedora Project
Vendor      : Fedora Project
$ dnf repoquery --installed --qf '%{name} from_repo=%{from_repo}' dracut-sshd
dracut-sshd from_repo=fedora
```

Using the distribution package (rather than COPR) means the module inherits Fedora's signing and
update path — relevant for a build whose pre-boot SSH surface must stay patched.

```
# Install the dracut-sshd module
sudo dnf install dracut-sshd          # installed: dracut-sshd-0.7.1-5.fc44.noarch

# Rebuild the initramfs to embed the SSH daemon and authorized key
sudo dracut -f --regenerate-all
```

## Dracut configuration

The initramfs is built with pre-boot networking (`systemd-networkd`), the `fido2` unlock module,
and `dracut-sshd`. The relevant `/etc/dracut.conf.d/*.conf` drop-in:

```
add_dracutmodules+=" systemd-networkd "
install_items+=" /etc/systemd/network/20-dracut-wired.network "
kernel_cmdline="rd.neednet=1"
add_dracutmodules+=" fido2 "
```

The running kernel command line (`/proc/cmdline`) — note `rd.neednet=1` forces the network up in
the initramfs, and the LUKS volume is referenced by UUID (there is no legacy `ip=` directive;
addressing is declarative via the embedded `20-dracut-wired.network`):

```
BOOT_IMAGE=(hd1,gpt2)/vmlinuz-7.0.11-200.fc44.x86_64 root=UUID=c1171ebd-167a-4ed7-a7c0-5b63d2e1b007 ro rootflags=subvol=root rd.luks.uuid=luks-6cbc50ba-6f8a-4932-abfc-f2d0504a29b3 rhgb quiet
```

- **`fido2` module present** → FIDO2 unlock is active *in the initramfs*, not only post-boot: a
  Nitrokey touch satisfies the LUKS prompt directly.
- **`systemd-networkd` + `20-dracut-wired.network`** → wired networking is configured before the
  root volume mounts, which is what makes the dracut-sshd remote-unlock path reachable.

## Key material

- A dedicated **ed25519** key pair is generated in Termux on the GrapheneOS device.
- The **public** key is baked into the initramfs `authorized_keys` at build time.
- This key is **separate** from all other SSH keys on the system and is rotated whenever the
  initramfs is rebuilt.
- The initramfs SSH **host key** is pinned on the GrapheneOS client to prevent MITM substitution
  during the unlock exchange.

## Unlock flow

1. The workstation powers on; `systemd-networkd` brings up the wired interface (`rd.neednet=1`)
   and reaches the initramfs SSH listener (pre-mount), pausing at the LUKS prompt.
2. The GrapheneOS handset (Termux) connects to the pre-boot listener **over Tailscale**. Tailscale
   itself does not run in the initramfs — the modules are `systemd-networkd` + `fido2` +
   `dracut-sshd`; reachability comes from the LAN the pre-boot interface sits on being exposed over
   the tailnet (subnet router / same-network client).
3. The connection authenticates against the embedded ed25519 public key (host key pinned on the
   client to prevent MITM substitution).
4. Inside the session, the password agent is signalled to drive the LUKS2 unlock over the encrypted
   channel:

   ```
   systemd-tty-ask-password-agent
   ```

5. A Nitrokey 3A NFC present at the machine is **touched** (the initramfs `fido2` module handles
   the FIDO2 unlock); the LUKS2 volume unlocks and boot continues into userspace. The two-factor
   property holds: remote network access (Tailscale + pinned ed25519 key) **and** physical presence
   (the touch on the hardware token).

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
| dracut-sshd in initramfs (v0.7.1-5.fc44) | Operational |
| `systemd-networkd` + `fido2` initramfs modules | Operational |
| Termux ed25519 key embedded | Operational |
| Remote unlock via `systemd-tty-ask-password-agent` | Tested working |
| Host-key pinning on client | Operational |

## Key rotation & initramfs rebuild

The initramfs is rebuilt whenever the embedded authorized key changes. To rotate the unlock key:

```
# 1. On the GrapheneOS handset (Termux): generate a fresh key pair
ssh-keygen -t ed25519 -f ~/.ssh/ironveil_unlock

# 2. On the workstation: replace the public key dracut-sshd embeds, then rebuild the initramfs
#    (dracut-sshd reads root's authorized_keys / its configured key path at build time)
sudo dracut -f --regenerate-all

# 3. On the handset: re-pin the new initramfs SSH host key on first connect (verify the
#    fingerprint out-of-band before trusting it)
```

Cross-reference: the handset side of this key custody is the nullbyte **Façade**-profile Termux
SSH integration (the unlock private key lives only in that profile).
