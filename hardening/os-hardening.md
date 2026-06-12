# OS-Level Hardening — MAC, syscall filtering, boot integrity

Living build — values captured from the running machine (war-horse) on 2026-06-12. This file
documents the host-level controls that sit *above* disk and network encryption: mandatory access
control, syscall filtering, and the honest state of UEFI boot integrity. Where a control is **not**
enabled, that is stated plainly — an accurate posture is worth more than an aspirational one.

## Mandatory Access Control — SELinux

SELinux is left in **enforcing** mode (Fedora ships it enforcing; it is deliberately *not*
disabled, which is a common and damaging shortcut on developer workstations).

```
$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      35
```

- **`targeted` policy, `enforcing`** at both runtime and in `/etc/selinux/config` — there is no
  reboot-to-permissive drift.
- **MLS status enabled** in the loaded policy — the multi-level constraints are compiled in even
  though the `targeted` policy is the active type-enforcement model.
- **`deny_unknown: allowed`** — unknown object classes are permitted rather than denied; this is the
  Fedora default and is noted here for completeness, not as a hardening claim.

Type enforcement confines the network-facing and pre-boot components of this build (the SSH path,
systemd-resolved, AdGuard) to their policy domains, so a compromise of one daemon does not grant
free rein over the host.

## Syscall filtering — seccomp

The running kernel is built with seccomp and the BPF filter mode, which is what systemd service
sandboxing (`SystemCallFilter=`, `RestrictNamespaces=`, etc.) depends on:

```
$ grep -i seccomp /boot/config-$(uname -r)
CONFIG_HAVE_ARCH_SECCOMP=y
CONFIG_HAVE_ARCH_SECCOMP_FILTER=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
```

`CONFIG_SECCOMP_FILTER=y` is the prerequisite for per-service syscall allow/deny lists. The
distribution's own unit files (and any hardened drop-ins) can therefore constrain the syscall
surface of long-running daemons. *Note (honest scope):* this records kernel-level capability;
a per-unit `systemd-analyze security <unit>` audit of individual service exposure is tracked as
`FUTURE WORK` rather than claimed here.

## ptrace restriction — Yama

```
$ cat /proc/sys/kernel/yama/ptrace_scope
0
```

`ptrace_scope = 0` is the Fedora default: a process may `ptrace` any other process of the same
UID. This is **not** the hardened setting. Raising it to `1` (restricted — only a parent may trace
a child) would reduce the blast radius of a single compromised same-user process attempting to read
another's memory (e.g. scraping an unlocked password manager).

> **FUTURE WORK — ptrace hardening:** set `kernel.yama.ptrace_scope = 1` via
> `/etc/sysctl.d/` and verify no developer tooling (debuggers attaching to running processes)
> breaks. Tracked, not yet applied — recorded here honestly rather than overstated.

## Boot integrity — UEFI Secure Boot (honest state)

```
$ mokutil --sb-state
SecureBoot disabled
Platform is in Setup Mode
```

**Secure Boot is currently disabled, and the platform is in Setup Mode** (no Platform Key
enrolled). This is an accurate statement of the present build, not the aspirational one:

- ironveil's confidentiality and unlock-integrity guarantees rest on **LUKS2 full-disk
  encryption** + **touch-only FIDO2 hardware-key unlock** + a **pinned initramfs SSH host key**,
  *not* on UEFI Secure Boot. An attacker who tampers with the bootloader still cannot decrypt the
  volume without a Nitrokey touch or the offline emergency passphrase.
- What Secure Boot *would* add is **pre-decryption tamper-evidence**: detection of an evil-maid
  bootloader/initramfs swap before the LUKS prompt is reached. The current build does not provide
  that, and does not claim to.

> **FUTURE WORK — boot integrity:** enrol a Platform Key and own keys (or shim+MOK), sign the
> kernel/initramfs, and move to Secure Boot enabled. Combined with TPM2 PCR sealing of a LUKS
> keyslot, this would close the evil-maid gap. See
> [research/luks2-fido2-reference.md](research/luks2-fido2-reference.md) §5 for the measured-boot
> design notes. Until then the gap is documented, not hidden.

## Summary

| Control | State | Hardened? |
|---------|-------|-----------|
| SELinux | `enforcing`, `targeted`, MLS compiled | Yes — left enforcing (not disabled) |
| seccomp | `CONFIG_SECCOMP_FILTER=y` (kernel support present) | Yes — available for unit sandboxing |
| Yama ptrace_scope | `0` (Fedora default) | No — tightening to `1` is FUTURE WORK |
| UEFI Secure Boot | disabled, Setup Mode (no PK) | No — FUTURE WORK; confidentiality rests on FDE+FIDO2 |

The two unhardened items are stated as gaps on purpose. A defence-in-depth write-up that only
lists wins is a marketing document; this one lists the next two moves as well.
