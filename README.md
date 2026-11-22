# mt3000-multiwan-bonding

Reproducible multi-WAN **MPTCP bonding on the GL.iNet GL-MT3000** (Beryl AX,
MediaTek MT7981B), intended for a TollGate whose uplinks are a 2.4 GHz mobile
hotspot and an open 5 GHz hotel guest network.

This repository records **what actually works, what does not, and which parts
are still missing upstream**. It is deliberately a status document first and an
automation repository second. The lab was partially brought up, the production
device was upgraded to the rc9 integration build, and the remaining bond test
is blocked by the airport network blocking the MPTCP server.

## Target topology

```text
mobile hotspot (2.4 GHz, ch5) ─┐
                               ├─ GL-MT3000 (TollGate) ─ guests
hotel guest WiFi (5 GHz, ch36) ┘
```

The MT3000 has two radios and one AP+STA channel per radio, so the only
workable layout is:

```text
radio0 (2.4 GHz): APs on ch5   + STA to the hotspot (ch5)
radio1 (5 GHz):   APs on ch36  + STA to the hotel guest (ch36)
```

The production device is `tollgate-76aa` (192.168.2.1); the lab DUT is the
`gl-mt3000` labgrid place on x280 (192.168.50.205).

## Status (2026-11-22)

### Works

| Capability | Evidence |
|---|---|
| Labgrid place `gl-mt3000` acquire/release | `labgrid-client -p gl-mt3000 acquire` on x280 |
| Device inventory | OpenWrt 25.12.5, kernel 6.12.94, TollGate v0.6.0 at start |
| Backup and off-device copy | `/etc/tollgate`, `/etc/config`, package list, sha256; 1.5 MB |
| Upgrade to OpenWrt 25.12.6 | kernel 6.12.112, config preserved, uptime verified |
| Build TollGate from `main` (v0.6.0-rc4) | `tollgate-wrt-0.6.0_rc4-r0.apk`, sha256 `15e41df2…` |
| Install the test APK | `tollgate version` = v0.6.0-rc4, commit 959280db |
| MPTCP kernel present | `net.mptcp.enabled=1` |
| `ip-full` present | `ip mptcp limits` returns `add_addr_accepted 0 subflows 2` |
| Proxy/runtime packages | `xray-core`, `kmod-wireguard`, `wireguard-tools`, `mptcpize` installed |
| Hotel guest visible from production device | 5 GHz ch36, about -78 dBm, open |
| Hotel association + DHCP | `10.32.77.42/20`, gateway/DNS `10.32.64.1` |
| ASK4 portal identified | `gw.customer.ask4.lan:3145/redirect.php`; no-field click-through → sign-up page (see [`docs/ASK4-PORTAL.md`](docs/ASK4-PORTAL.md)) |
| Hotspot visible from production device | 2.4 GHz ch5, about -42 dBm, WPA2/WPA3 transition |
| Production `tollgate-76aa` on rc9 | `0.6.0-rc9`, commit `eb879697`; wallet 0 sats |
| Production internet via `c03rad0r` | `sae-mixed`, 10.110.131.142, gateway 10.110.131.143 (hotel test) |
| Airport uplink `_FreeWifi` | Manchester Airport, 5 GHz, `10.250.73.81/23`, gateway `10.250.73.254`; Purple portal completed; `ifconfig.me` = 62.233.55.124 |
| Airport free plan | MAG MAN FREE Wi-Fi, 2 hours / 2 GB / 7.5 Mbps; tied to the STA MAC `96:83:c4:8c:59:c2` |
| Airport MPTCP server | **unreachable on every port**; two-path bond cannot form on this network (see [`docs/AIRPORT-TEST.md`](docs/AIRPORT-TEST.md)) |
| ASK4 guest login | `/en/guest-signup` first/last name → hotel site; hotel STA MAC then had internet (public IP 31.205.35.59) |
| MPTCP second-path mechanism | manual endpoints + metrics 10/20 + per-source routes → hotel `SF_ESTABLISHED` against an MPTCP echo server |
| **Controlled two-path bond** | VLESS/BSBF xray `:16384` + `tcpMPTCP`; both subflows `SF_ESTABLISHED`; 4 MB at ~37.7 Mbit/s; hotel carried ~4.13 MB, c03rad0r ~159 KB (see [`docs/MPTCP-BONDING-TEST.md`](docs/MPTCP-BONDING-TEST.md)) |
| Persistent production bond | worked on rc8 with radio1 dedicated to the hotel STA and both subflows; **disabled at the airport** because the MPTCP server is unreachable. The airport uplink is instead the normal default route. |

### Does not work / not yet verified

| Item | Why |
|---|---|
| Two simultaneous STA uplinks through the product | The upstream manager is single-STA (`disableOtherSTAInterfaces`, `CleanupStaleSTAs`, one `wwan`). Manual UCI was written, but association/DHCP verification was blocked by the lab outage. |
| Guest traffic entering the bond | No NDS-aware steering existed in `main`; PR #1258 implements it, not merged/tested. |
| Per-uplink policy routing | Missing in `main`; PR #1254 implements it, not merged/tested. |
| NAT-safe transport through `tollgate config` | `transport`/`tunnels` were not exposed; PR #1255 fixes it, not merged. |
| `ip-full` on a stock install | Not declared in the package; PR #1256 fixes it, not merged. |
| Captive-portal auto-login | PR #1253 implements uplink-scoped click-through/form login, allow-listed redirects and a session budget. It handles a single form; Manchester Airport's Purple flow is multi-step (email registration → plan → controller `cmd=authenticate`) and was completed manually. |
| Multi-step portal recipes | The current recipe schema has no ordered steps, so Purple/Aruba cannot be automated yet. |
| Airport two-path bond | The airport blocks `167.235.218.44` on every tested port, so the second subflow has no far end. |
| Policy enum through the CLI | Schema said `always`, bond validates `all`; PR #1257 fixes it. |
| Persistent bond caveats | The client uses the server’s static VLESS UUID (not a paid session); the ASK4 first/last-name login is replayed by a cron watcher, not the upstream per-uplink portal code; uplink addresses/gateways are static; guest TPROXY steering is installed (`mptcp_steer`, :1082) but not yet exercised by a real guest (the image has no `veth` for a synthetic test). |
| ASK4 persistent account | The guest login authorized the MAC with just first/last name; no persistent ASK4 account/email verification was created. The `Create Account` path (`/en/speed`) returned HTTP 500. |
| `bsbf-mptcp` helper | Calls `ip mp e a ...`, which fails on ip-full 6.18; manual `ip mptcp endpoint add` works. |
| Production bonding enabled | rc6 is installed with bonding disabled; the working uplink is `c03rad0r`. |

### Lab blocker

At about 00:30 on 2026-11-22 the x280 FIPS link and the MT3000 behind it
stopped answering. `tollgate-76aa` is still reachable on 192.168.2.1, but it
cannot reach the MT6000 (192.168.2.2) or the MT3000. No code or device change
caused this; the lab path is down. The resume procedure is in
[`docs/ACCESS.md`](docs/ACCESS.md).

## Upstream gaps and PRs

All against [`OpenTollGate/tollgate-module-basic-go`](https://github.com/OpenTollGate/tollgate-module-basic-go):

| PR / issue | What it changes |
|---|---|
| [#1257](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1257) | policy enum `all` (was `always` in schema) |
| [#1256](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1256) | declare `ip-full`; apk/opkg runbook |
| [#1255](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1255) | expose `transport` and `tunnels` in config |
| [#1254](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1254) | per-uplink policy routing + disable flush |
| [#1258](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1258) | NDS-aware steering (no pre-auth bypass) |
| [#1253](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1253) | automatic captive-portal login; uplink-scoped with allow-list, cookie jar, watcher, session budget, and CLI redaction for the portal secrets |
| [#1261](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1261) | protect bonded STA sections from the single-STA manager |
| [#1262](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1262) | pass the uplink address (not the gateway) to the bond |
| [#1263](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1263) | periodic bond drift reapply (`reconcile_interval_s`) |
| [#1264](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1264) | per-AP consent manifest before joining open upstreams |
| [#1259](https://github.com/OpenTollGate/tollgate-module-basic-go/issues/1259) | design: two concurrent STA uplinks |
| [#1260](https://github.com/OpenTollGate/tollgate-module-basic-go/pull/1260) | MT3000 hardware/CVE/two-STA documentation |

See [`docs/TOLLGATE-GAPS.md`](docs/TOLLGATE-GAPS.md) for the code references
and why each gap matters.

### Combined integration branch

All seven PRs are merged with conflicts resolved on
`c03rad0r/tollgate-module-basic-go` branch `integration/mt3000-bond`
(commit `a4c2462b`, including the ASK4 `SO_MARK`/venue-DNS fix). Combined test
gate:

```text
config_manager        ok
mptcp_bonding         ok
portal targeted tests ok
root build             ok
packaging fences       7 passed / 34 passed
```

Installable test artifact (not yet installed; lab offline):

```text
tollgate-wrt-0.6.0_rc6-r0.apk
sha256 86c2a1736263c3f654fcc2e8b20de347fd6b6782bce5d812a7cfb04e47d19414
```

The branch is for integration testing only; the reviewable units remain the
seven PRs above.

## Safety

- 25.12.6 ships kernel **6.12.112**, still below the CVE-2026-98432 fix floor
  (6.12.113). The lab is isolated; the production hotel device needs a patched
  kernel or an explicit risk decision.
- No WiFi PSKs, portal passwords, wallet seeds, or Cashu tokens are stored in
  this repository. Device backups stay off-repo.
- The production device's wallet was 0 sats at inspection.

## Repository layout

```text
docs/TOPOLOGY.md        radio/channel plan and AP+STA constraints
docs/TOLLGATE-GAPS.md   what is missing upstream, with code references
docs/ACCESS.md          labgrid/SSH access and the resume procedure
docs/MEASUREMENTS.md    throughput/failover record (empty until lab returns)
ansible/                provisioning skeleton (not yet run)
scripts/                verification/measurement helpers
evidence/               sanitized logs and command output
```

## Licence

MIT for this repository. The TollGate module is GPL-3.0; xray-core is MPL-2.0;
kernel MPTCP and iproute2 are upstream.
