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

## Resolution checklist
- [ ] crypttab line pasted → luks2-setup.md:68 updated, marker removed
- [ ] luksDump cipher/keysize → luks2-setup.md:12 updated
- [ ] dracut drop-in + cmdline → dracut-sshd.md:27 updated; package source → dracut-sshd.md:19
- [ ] WireGuard excerpt (secrets redacted) → network-stack.md §WireGuard; kill-switch → :49
- [ ] AdGuard upstream + blocklists → network-stack.md :27 / :20
- [ ] DNS-leak result → network-stack.md:79; Fedora/kernel → luks2-setup.md:88
- [ ] Re-run the privacy / handle-typo scan, commit per the standing convention, push.
