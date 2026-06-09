# AdGuard Home + WireGuard + systemd-resolved — Network Stack Reference

Technical reference for the DNS/VPN chain described in
[../network-stack.md](../network-stack.md):

```
application → systemd-resolved (127.0.0.1) → AdGuard Home → WireGuard tunnel → upstream (10.2.0.1)
```

Reference document explaining the cryptography and the leak-prevention properties of this chain.

---

## 1. AdGuard Home architecture

AdGuard Home is a self-hosted recursive-forwarding DNS server with filtering. In IRONVEIL it runs
locally, bound to `127.0.0.1:53`, and forwards upstream over the WireGuard tunnel.

### 1.1 Roles it plays

- **Local resolver / forwarder.** It receives queries from `systemd-resolved` on loopback, applies
  filtering, and forwards permitted queries to its configured upstream(s).
- **Filtering engine.** Block/allow lists (tracking, malware, telemetry) and custom rules are
  evaluated per query. A blocked name is answered with `NXDOMAIN`/`0.0.0.0`/a null response
  depending on rule type — the query never leaves the host.
- **Query log / metrics.** Local visibility into what every process resolved — useful for a
  security practitioner who wants to see beaconing/telemetry from their own tooling.

### 1.2 Encrypted-transport options (DoH / DoT / DoQ)

AdGuard Home can both *speak* and *serve* encrypted DNS:

- **DNS-over-HTTPS (DoH, RFC 8484):** DNS wire-format wrapped in HTTPS (port 443). Blends with
  normal web traffic, hardest to block/distinguish on the wire.
- **DNS-over-TLS (DoT, RFC 7858):** DNS over a dedicated TLS connection (port 853). Easy to
  identify by port, simple to deploy.
- **DNS-over-QUIC (DoQ, RFC 9250):** DNS over QUIC/UDP 853 — lower latency, avoids TCP
  head-of-line blocking.

In the IRONVEIL chain the *upstream* hop is to `10.2.0.1`, a WireGuard peer reached **through the
encrypted tunnel**. So the confidentiality of the upstream query is already provided by WireGuard;
whether AdGuard additionally uses DoH/DoT to that peer is a defence-in-depth choice. *Verify
whether the upstream is plain DNS over the tunnel or DoH/DoT over the tunnel, and record it.*

### 1.3 Upstream chaining

AdGuard Home supports multiple upstream modes:

- **Parallel/fastest:** query several upstreams, take the fastest answer (lower latency, more
  metadata exposure to multiple resolvers).
- **Sequential/fallback:** try in order.
- **Conditional (domain-specific upstreams):** route `example.internal` to one resolver and
  everything else to another — the `[/domain/]upstream` syntax. Useful for split-horizon
  (internal names resolved differently from public ones).

IRONVEIL points the upstream at the single WireGuard-peer resolver, so all permitted queries exit
via the tunnel to `10.2.0.1`.

---

## 2. Security value of local DNS filtering

### 2.1 Threat classes it mitigates

- **Tracking/telemetry exfiltration via DNS.** Apps and OS components that phone home by name are
  blocked at resolution. For a practitioner this also reveals which of their own tools are chatty.
- **Known-malicious domains.** Malware C2, phishing, and malvertising domains on the block lists
  are denied resolution, breaking the name-resolution step of those kill chains.
- **First-hop metadata leakage to the LAN/ISP.** Binding the resolver to loopback and sending
  upstream through WireGuard means the LAN and ISP never see plaintext queries (see §5).
- **Ad/malvertising surface reduction.** Fewer third-party domains resolved = smaller drive-by
  surface.

### 2.2 What it does **not** mitigate (honest limits)

- **Encrypted-SNI-less TLS still leaks the destination.** Even if a name resolves through the
  tunnel, once a connection is made the destination IP is visible to the network path, and TLS SNI
  (without ECH) leaks the hostname. DNS filtering is not traffic confidentiality.
- **Hardcoded IPs / DoH-bypassing malware.** Malware that connects to a raw IP or runs its own DoH
  resolver bypasses the local resolver entirely. A firewall egress policy is needed to force all
  DNS through AdGuard and block rogue DoH.
- **Domain fronting / fast-flux / DGA.** Block lists lag novel domains; DGA-based C2 generates
  domains faster than lists update.
- **Content, not just names.** A permitted domain can still serve malicious content. DNS filtering
  is a coarse, name-level control — one layer, not the whole defence.
- **Compromised upstream/lists.** Trust is shifted to the upstream resolver and the list
  maintainers.

---

## 3. WireGuard cryptographic architecture

WireGuard is a minimal, modern VPN built on a fixed, opinionated cryptographic suite ("crypto
versioning" rather than negotiation), which removes the downgrade/negotiation attack surface that
plagues IPsec/OpenVPN.

### 3.1 The Noise protocol framework

WireGuard's handshake is an instantiation of the **Noise framework**, specifically the
**`Noise_IKpsk2`** pattern:

- **IK:** the initiator knows the responder's static public key in advance (Immediate, with the
  responder's Known static key). Both sides have static Curve25519 identity keys; ephemeral keys
  are generated per handshake.
- **psk2:** an *optional* pre-shared symmetric key mixed in as a second factor — this is the field
  used for post-quantum hardening (a shared secret an attacker would also need even if they broke
  the Curve25519 ECDH). *Verify whether IRONVEIL configures a PSK on `wg-SE-RO-1`.*
- The handshake provides mutual authentication, forward secrecy (via the per-session ephemeral
  keys), and identity hiding for the initiator.

### 3.2 The primitives

| Function | Primitive |
|----------|-----------|
| Key exchange | **Curve25519** ECDH (X25519) |
| Authenticated encryption (data) | **ChaCha20-Poly1305** (AEAD, RFC 8439) |
| Hashing | **BLAKE2s** |
| Key derivation | HKDF (built on BLAKE2s) |
| Handshake MAC / DoS mitigation | Keyed-BLAKE2s (cookie mechanism) |

- **ChaCha20-Poly1305** is a stream-cipher-based AEAD: fast in software (no AES-NI dependency),
  constant-time, and resistant to the timing side-channels that table-based AES can suffer on
  hardware without AES instructions. Each data packet is encrypted under a per-session key with a
  64-bit counter nonce.
- **Curve25519** gives ~128-bit security with no fragile parameter choices; it underpins the
  static identity keys and the ephemeral handshake keys (forward secrecy: compromise of a static
  key does not expose past sessions whose ephemeral keys are gone).
- Sessions re-key roughly every 2 minutes / after a volume of data, limiting the material protected
  by any one session key.

### 3.3 Why the fixed suite matters

Because there is no cipher negotiation, there is no downgrade attack and no large parser to exploit
(WireGuard's codebase is famously small). The cost is that "upgrading" crypto means a protocol
revision rather than a config toggle — an acceptable trade for the reduced attack surface.

---

## 4. NetworkManager integration vs `wg-quick`

IRONVEIL manages the tunnel `wg-SE-RO-1` as a **NetworkManager** connection rather than via
`wg-quick`.

| Aspect | `wg-quick` | NetworkManager |
|--------|-----------|----------------|
| Definition | `/etc/wireguard/<iface>.conf`, brought up by a oneshot service | A NM connection profile (`nmcli`/keyfile) |
| Lifecycle | Manual or a simple systemd unit | First-class NM connection: autoconnect, per-network policy, roaming |
| DNS handling | `DNS=` line invokes `resolvconf`/`resolvectl` hooks | Integrates with `systemd-resolved` natively (per-link DNS) |
| Roaming | Re-up on network change is manual | NM reacts to link changes and can re-establish |
| Kill-switch | Via `PostUp`/`PreDown` firewall rules and `Table` routing tricks | Via NM connection properties + firewall/route policy |

### 4.1 Operational differences that matter here

- **DNS as a first-class property.** Because the tunnel is an NM connection, its DNS can be bound
  to the link and ordered correctly with `systemd-resolved`, which is central to leak prevention
  (§5). With `wg-quick` the `DNS=` directive shells out to set resolvers, which is more fragile on a
  `systemd-resolved` system.
- **Persistence / autoconnect.** NM can bring the tunnel up automatically and keep it up across
  roaming, rather than depending on a separately-enabled `wg-quick@` unit.
- **Readability of multi-tunnel setups.** Named connections (`wg-SE-RO-1` encoding region/role)
  are managed uniformly via `nmcli`.

### 4.2 Kill-switch

The goal of a kill-switch is: **if the tunnel drops, no traffic egresses in the clear** (no DNS,
no app traffic). Implementations:

- **Routing-based (wg-quick style):** `AllowedIPs = 0.0.0.0/0, ::/0` plus a separate routing table
  and a `fwmark` so that any packet not going through the tunnel is dropped. wg-quick automates
  this with its `Table`/`PostUp` rules.
- **Firewall-based:** an `nftables`/`firewalld` policy that permits egress only on the `wg-SE-RO-1`
  interface (and to the WireGuard endpoint itself), dropping everything on the physical interface
  except what is needed to (re)establish the tunnel.
- **NM dispatcher script:** a script in `/etc/NetworkManager/dispatcher.d/` that reacts to the
  tunnel going down and tears down/blocks the default route.

The build note records the kill-switch implementation choice as a TODO. **Recommended:** an
explicit `nftables` egress-lock keyed to the WireGuard interface is the most auditable approach and
is independent of routing-table subtleties. *Open item: confirm and document which is in use.*

---

## 5. DNS-leak analysis

### 5.1 How leaks happen with VPN + local DNS

A "DNS leak" is when a name query egresses **outside** the intended encrypted path — revealing
browsing to the LAN/ISP/another resolver even though application traffic is tunnelled. Common
causes:

1. **Resolver not bound to the tunnel.** The OS keeps using a DHCP-provided LAN resolver, so
   queries go to the ISP in plaintext while only app traffic uses the VPN.
2. **systemd-resolved split-DNS / per-link servers.** With multiple links, `resolved` can send some
   queries to a non-tunnel link's resolver unless DNS is forced to the intended path.
3. **Plain stub listener exposure.** A stub on a routable address (not loopback) can be queried off
   host.
4. **App-level DoH bypass.** A browser with its own DoH resolver ignores the system resolver
   entirely.
5. **IPv6 leakage.** IPv6 queries/traffic escaping a v4-only tunnel config.

### 5.2 How the IRONVEIL chain prevents them

```
application
   │  (libc / nss → 127.0.0.1)
   ▼
systemd-resolved   DNS=127.0.0.1, Domains=~., DNSStubListener=no
   │  ~. routes ALL domains to the single configured server
   ▼
AdGuard Home  (bound 127.0.0.1:53 — never leaves host unfiltered)
   │  upstream = 10.2.0.1
   ▼
WireGuard wg-SE-RO-1  (query encrypted inside the tunnel)
   ▼
upstream resolver 10.2.0.1  (sees the tunnel exit IP, never the host IP, never plaintext on the LAN)
```

The specific properties that close each leak vector:

- **`Domains=~.`** makes `systemd-resolved` send *every* domain to the configured server
  (`127.0.0.1`), defeating split-DNS routing to a LAN link's resolver (vector 2).
- **`DNS=127.0.0.1`** forces the upstream to AdGuard, not a DHCP-supplied resolver (vector 1).
- **`DNSStubListener=no`** disables `resolved`'s own `:53` stub so there is no second listener and
  no off-host exposure; AdGuard owns `127.0.0.1:53` (vectors 3, and avoids a port clash).
- **AdGuard bound to loopback** means no unfiltered DNS ever leaves the host; the only egress is
  AdGuard's upstream, which goes **through WireGuard** (vector 1/3).
- **WireGuard tunnelling** of the upstream hop means the LAN/ISP see only ChaCha20-Poly1305
  ciphertext, and the upstream resolver sees the tunnel exit address — not the host.

### 5.3 Residual leak risks to verify

- **IPv6 (vector 5):** confirm IPv6 is either fully routed through the tunnel or disabled, and that
  `resolved` is not handing out a v6 link resolver. Otherwise v6 queries can bypass the v4 path.
- **App-embedded DoH (vector 4):** browsers/apps with their own DoH must be disabled or their DoH
  endpoints blocked at the firewall, or they bypass AdGuard entirely.
- **Pre-tunnel boot window:** before `wg-SE-RO-1` is up, AdGuard's upstream is unreachable —
  confirm the kill-switch prevents fallback to a plaintext LAN resolver during that window.

### 5.4 Verification

```bash
resolvectl status                      # DNS=127.0.0.1 active on the right link; no LAN resolver listed
sudo ss -ulpn 'sport = :53'            # only AdGuard bound to 127.0.0.1:53; resolved stub absent
resolvectl query example.com           # resolves via 127.0.0.1 → AdGuard
wg show wg-SE-RO-1                      # tunnel up, recent handshake, expected peer/endpoint
# external DNS-leak test service → should show only the WireGuard exit, no ISP resolver
# IPv6 check: ensure no v6 resolver is offered and v6 egress is tunnelled or disabled
```

---

## 6. Open items / to verify

- [ ] Whether the upstream hop to `10.2.0.1` is plain DNS over the tunnel or DoH/DoT over the
      tunnel (defence-in-depth).
- [ ] Whether `wg-SE-RO-1` uses a WireGuard PSK (`Noise_IKpsk2` second factor / PQ hardening).
- [ ] The exact kill-switch implementation (nftables egress-lock recommended) and its boot-window
      behaviour.
- [ ] IPv6 handling end-to-end (tunnelled or disabled; no v6 resolver leak).
- [ ] Whether app-embedded DoH is blocked at the firewall.
- [ ] The exact block lists enabled in AdGuard (enumerate in the build notes).

> No vendor performance claims, CVEs, or specific list contents are asserted here without
> verification. RFC numbers cited (8484 DoH, 7858 DoT, 9250 DoQ, 8439 ChaCha20-Poly1305, 9106
> Argon2) are stable references; confirm any that you reuse in published material.
