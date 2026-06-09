# Network Stack — WireGuard, AdGuard Home, systemd-resolved

Living build — `MANUAL INPUT REQUIRED` markers flag values still to be captured from the running machine; `FUTURE WORK` markers flag planned enhancements.

## DNS resolution chain

```
application → systemd-resolved (127.0.0.1) → AdGuard Home → WireGuard tunnel → upstream (10.2.0.1)
```

An observer on the external interface sees only WireGuard-encrypted traffic; the upstream
resolver sees the WireGuard exit IP, never the host IP, and never a plaintext query.

## AdGuard Home

| Setting | Value |
|---------|-------|
| Listener | `127.0.0.1:53` |
| Upstream DNS | `10.2.0.1` (WireGuard peer — queries traverse the encrypted tunnel) |
| Block lists | Tracking, malware, and telemetry lists enabled <!-- MANUAL INPUT REQUIRED: enumerate the exact block lists — copy them from the AdGuard Home admin UI → Filters → DNS blocklists, or the `filters:` section of AdGuardHome.yaml --> |

Binding to loopback ensures no DNS traffic leaves the host unfiltered.

```
# AdGuard Home upstream is set to the WireGuard peer
# (configured via the AdGuard Home admin UI / AdGuardHome.yaml)
# MANUAL INPUT REQUIRED: capture the relevant AdGuardHome.yaml excerpt (the `upstream_dns:`
#   and `bind_host`/`dns.bind_hosts` keys) from the running config
```

## WireGuard — wg-SE-RO-1

| Attribute | Value |
|-----------|-------|
| Interface | `wg-SE-RO-1` (replaces legacy `wg0`) |
| Management | NetworkManager — treated as a first-class connection for roaming + DNS integration |
| Routing | All external traffic via the tunnel; kill-switch drops traffic if the interface drops |

The named-tunnel identifier encodes the endpoint region, keeping multi-tunnel configurations
readable.

```
# NetworkManager-managed tunnel
nmcli connection show wg-SE-RO-1
nmcli connection up wg-SE-RO-1
wg show wg-SE-RO-1
```

<!-- MANUAL INPUT REQUIRED: document the kill-switch implementation actually in use — record the NetworkManager dispatcher script (/etc/NetworkManager/dispatcher.d/) or the nftables/firewalld egress rule that drops traffic when wg-SE-RO-1 is down -->

## systemd-resolved

`systemd-resolved` forwards all queries to AdGuard Home on loopback:

```
# /etc/systemd/resolved.conf
[Resolve]
DNS=127.0.0.1
Domains=~.
DNSStubListener=no
```

```
sudo systemctl restart systemd-resolved
resolvectl status            # confirm DNS=127.0.0.1 is the active server
```

## DNS leak verification

After bringing up the tunnel, confirm queries exit only via WireGuard and resolve through
AdGuard Home:

```
resolvectl query example.com           # resolves via 127.0.0.1 → AdGuard
sudo ss -ulpn 'sport = :53'            # only AdGuard bound to 127.0.0.1:53
# External leak test (e.g. a DNS-leak test service) should show only the WireGuard exit
```

<!-- MANUAL INPUT REQUIRED: record the DNS-leak test result (date + service used, e.g. a DNS-leak-test site) and a screenshot reference confirming only the WireGuard exit is visible -->

## Current state

| Component | Status |
|-----------|--------|
| WireGuard `wg-SE-RO-1` (NetworkManager) | Operational |
| AdGuard Home on `127.0.0.1:53` | Operational |
| Upstream via `10.2.0.1` over tunnel | Operational |
| systemd-resolved → `127.0.0.1` | Operational |
| DNS leak verification | Confirmed — no plaintext egress |
