# IRONVEIL MANUAL INPUTS — COMPLETE BEFORE APPLICATIONS

> Single-session resolution guide for the hardware values still marked `MANUAL INPUT REQUIRED`
> across the ironveil docs. Run each command **on the ironveil workstation (war-horse)**, paste the
> output into the matching block, then transcribe it into the referenced file + line and delete the
> `MANUAL INPUT REQUIRED` marker there. Target: one ~5-minute pass with the machine in front of you.
>
> **Sanitisation rule (applies throughout):** never paste private keys, preshared keys, or real
> public-key material into a public repo. Redact secrets as `[REDACTED]`; keep only structural
> config (interfaces, peers, endpoints, AllowedIPs, cipher names, list names).

---

### 1. crypttab entry
**What this enables:** confirms the real at-rest cryptography (cipher, key size, active keyslots)
and the live unlock options — turning the documented LUKS2/Argon2id defaults into machine-attested
fact a reviewer can trust. **Expected format:** one `crypttab` line; `luksDump` reporting
`aes-xts-plain64`, 512-bit key, 3 active keyslots.

Run on war-horse:
```
cat /etc/crypttab
```
Paste output below:
```
[PASTE HERE]
```
Update: `hardening/luks2-setup.md` — **crypttab options** placeholder at **line 68** (the
`<!-- MANUAL INPUT REQUIRED: confirm the final crypttab options ... -->` comment in the
"crypttab configuration" section, line 58 onward).

Also capture the LUKS cipher/keysize while you have the disk open (resolves the line-12 placeholder):
```
sudo cryptsetup luksDump /dev/sda3        # read Data segments → cipher (expect aes-xts-plain64) + key size (expect 512-bit); confirm 3 active keyslots
```
Update: `hardening/luks2-setup.md` — **cipher** placeholder at **line 12**.

---

### 2. dracut-sshd config
**What this enables:** proves the pre-boot remote-unlock network path (the standout control) is real
and reproducible, and pins the exact package source + version for auditability. **Expected format:**
the `dracut.conf.d` drop-in lines + the `ip=` directive from `/proc/cmdline`; COPR repo name +
`rpm -q dracut-sshd` version string.

Run on war-horse:
```
cat /etc/dracut.conf.d/*.conf
cat /proc/cmdline                          # the ip= directive + network module on the running kernel
```
Paste relevant lines below:
```
[PASTE HERE]
```
Update: `hardening/dracut-sshd.md` — **config drop-in / network setup** placeholder at **line 27**.
While here, pin the package source + version (resolves the line-19 placeholder): record the COPR
repo and the installed `dracut-sshd` version (`rpm -q dracut-sshd` / `dnf info dracut-sshd`).

---

### 3. WireGuard config excerpt (sanitised)
**What this enables:** substantiates the named-tunnel (`wg-SE-RO-1`) and kill-switch claims with real,
secret-redacted config — showing egress is genuinely tunnel-bound rather than asserted. **Expected
format:** `[Interface]`/`[Peer]` sections with `Endpoint`, `AllowedIPs`, interface name; `PrivateKey`
and `PresharedKey` shown as `[REDACTED]`; plus the dispatcher/nftables kill-switch rule.

Run on war-horse:
```
sudo cat /etc/NetworkManager/system-connections/wg-*.nmconnection
```
Remove: `PrivateKey`, `PresharedKey` values (replace each with `[REDACTED]`).
Keep: `[interface]` / `[peer]` sections, `Endpoint`, `AllowedIPs`, interface name `wg-SE-RO-1`.
Paste sanitised excerpt below:
```
[PASTE HERE]
```
Update: `hardening/network-stack.md` — **WireGuard — wg-SE-RO-1** section (**line 31** onward).
Also document the kill-switch actually in use (resolves the line-49 placeholder): the
NetworkManager dispatcher script (`/etc/NetworkManager/dispatcher.d/`) or the nftables/firewalld
egress rule that drops traffic when `wg-SE-RO-1` is down.

> Path note: the template's `network/wireguard.md` does not exist in this repo — WireGuard lives in
> `hardening/network-stack.md`. Use that file.

---

### 4. AdGuard Home DNS upstream
**What this enables:** proves DNS is filtered and bound to loopback before egress (no plaintext leak),
backing the DNS-security layer with the actual upstream + blocklists in use. **Expected format:** the
`upstream_dns:` value (confirm `10.2.0.1` over the tunnel) and the enabled blocklist *names* only — no
auth tokens; plus the DNS-leak-test result (date + service).

From the AdGuard Home admin UI: **Settings → DNS settings → Upstream DNS** (and **Filters → DNS
blocklists**). Equivalent on disk: the `upstream_dns:` and `filters:` sections of `AdGuardHome.yaml`.
Paste the upstream config + enabled blocklist names below (no auth tokens):
```
[PASTE HERE]
```
Update: `hardening/network-stack.md` — **AdGuard Home** section: upstream `upstream_dns:` excerpt
placeholder at **line 27**, and the **block-list enumeration** placeholder at **line 20**. Confirm
the upstream is `10.2.0.1` over the tunnel (line 19).

> Path note: the template's `network/adguard.md` does not exist — AdGuard config also lives in
> `hardening/network-stack.md`.

Optional, same session (clears the remaining network-stack placeholders):
```
resolvectl query example.com              # confirm resolution via 127.0.0.1 → AdGuard
sudo ss -ulpn 'sport = :53'               # confirm only AdGuard bound to 127.0.0.1:53
```
Plus a DNS-leak test result (date + service used) for the **line-79** placeholder, and
`cat /etc/fedora-release && uname -r` for the Fedora/kernel placeholder (luks2-setup line 88).

---

## Resolution checklist — RESOLVED 2026-06-11 (operator hardware session)
- [x] crypttab line pasted → luks2-setup.md updated, marker removed (UUID `6cbc50ba-…`, `discard,x-initrd.attach,fido2-device=auto`)
- [x] luksDump cipher/keysize → luks2-setup.md updated (aes-xts-plain64, 512-bit, 3 keyslots: Argon2id + 2× PBKDF2/SHA-512 FIDO2)
- [x] dracut drop-in + cmdline → dracut-sshd.md updated (systemd-networkd + fido2 modules, rd.neednet=1); package version → `dracut-sshd-0.7.1-5.fc44.noarch` (repo source still to confirm via `rpm -qi`)
- [x] WireGuard excerpt (secrets redacted) → network-stack.md §WireGuard; kill-switch documented honestly as route-based/implicit (no fail-closed rule)
- [x] AdGuard upstream + blocklists → network-stack.md (Quad9 DoH upstream, AdGuard DNS filter, `*:53`)
- [x] DNS-leak result → network-stack.md (2026-06-11, via `wg-CH-FI-2`); Fedora/kernel → luks2-setup.md (Fedora 44, 7.0.11-200.fc44)
- [x] Privacy / handle-typo scan re-run, committed + pushed per the standing convention

### Corrections discovered this session (the docs were wrong, the hardware is ground truth)
- **Tunnel names:** the documented `wg-SE-RO-1` does not exist. Real tunnels: `wg-CH-FI-2` and
  `wg-SE-FI-1`. Fixed across hardening/, README.md, docs/index.html, and the research reference.
- **DNS upstream:** the documented `10.2.0.1` is the VPN provider's pushed DNS, **overridden** by
  AdGuard's real upstream **Quad9 DoH** (`https://dns10.quad9.net/dns-query`).
- **AdGuard listener:** real bind is `*:53` (all interfaces, pid 1452), not loopback-only — the
  no-plaintext-egress property rests on the DoH-over-tunnel upstream, documented honestly.
- **Kill-switch:** route-based (full-tunnel `AllowedIPs`), not a fail-closed rule; tunnels are
  `autoconnect=false` (manual). Flagged a hard kill-switch as FUTURE WORK.

### Optional values RESOLVED 2026-06-12 (checked from war-horse — the ironveil host; LUKS UUID prefix `6cbc50ba` matched)
- [x] **SELinux/seccomp status** → `hardening/os-hardening.md`: SELinux `enforcing`/`targeted`, MLS
  compiled; seccomp `CONFIG_SECCOMP_FILTER=y`; Yama `ptrace_scope=0` (Fedora default, flagged as
  FUTURE WORK to raise to 1). README marker resolved + Results rows added.
- [x] **Secure Boot state** → `mokutil --sb-state` = **disabled / Setup Mode** (no PK enrolled).
  Documented honestly in `os-hardening.md` + README: confidentiality rests on FDE+FIDO2, not Secure
  Boot; SB enrolment + TPM2 PCR sealing tracked as FUTURE WORK to close the evil-maid gap.
- [x] **dracut-sshd package origin** → **official Fedora repo** (Vendor/Packager Fedora Project;
  `from_repo=fedora`), not COPR. `hardening/dracut-sshd.md` updated.
- [x] **OpenRGB udev rule path** → `/usr/lib/udev/rules.d/60-openrgb.rules` (OpenRGB 0.9+ git).
  README marker resolved.
- [x] **Fedora/kernel re-verify** → Fedora 44, `7.0.11-200.fc44.x86_64`, cmdline `rootflags=subvol=root`
  (Btrfs) + `rd.luks.uuid=luks-6cbc50ba…` — matches documented, **no drift**.

### Still genuinely pending (need hardware interaction / operator)
- LUKS2 unlock-latency benchmark (hardware-key vs passphrase) — needs a timed reboot.
- OpenRGB saved **profile name** + colour scheme — needs the OpenRGB GUI (udev path now known).
- LUKS2 verified-boot hash — N/A while Secure Boot is disabled (no UEFI measured-boot chain to hash);
  becomes relevant only after the Secure Boot + TPM2 FUTURE WORK above is done.
