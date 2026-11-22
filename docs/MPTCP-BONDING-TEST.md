# Two-path MPTCP test on production (2026-11-22)

Device: `tollgate-76aa`, OpenWrt 25.12.5, TollGate **0.6.0-rc6** (a4c2462b).
Server: `mptcp-hel1` (`167.235.218.44`, SSH 65222), Debian ARM64, kernel
**6.12.113**, MPTCP limits 8/8.

## Paths

| Path | Interface | Address | Gateway | Status |
|---|---|---|---|---|
| c03rad0r | `phy0-sta0` | 10.110.131.142 | 10.110.131.143 | working WPA3 (`sae-mixed`) |
| Pendulum Hotel Guest | `phy1-sta0` | 10.32.76.220 | 10.32.64.1 | ASK4 guest login authorized, internet verified |

The hotel guest login was the ASK4 `/en/guest-signup` first-name/surname form;
it redirected to `pendulumhotel.co.uk` and the STA MAC then had internet
(`ifconfig.me` = 31.205.35.59). No email/room/password field was used.

## What worked

With manual endpoints, two main-table defaults with metrics, and per-source
routing:

```text
ip mptcp limits set subflow 2 add_addr_accepted 2
ip mptcp endpoint add 10.110.131.142 dev phy0-sta0 subflow fullmesh
ip mptcp endpoint add 10.32.76.220   dev phy1-sta0 subflow fullmesh
ip route replace default via 10.110.131.143 dev phy0-sta0 metric 10
ip route replace default via 10.32.64.1     dev phy1-sta0 metric 20
ip route replace 10.110.131.0/24 dev phy0-sta0 table 101
ip route replace default via 10.110.131.143 dev phy0-sta0 table 101
ip route replace 10.32.64.0/20 dev phy1-sta0 table 102
ip route replace default via 10.32.64.1 dev phy1-sta0 table 102
ip rule add from 10.110.131.142/32 lookup 101 pref 100
ip rule add from 10.32.76.220/32   lookup 102 pref 101
```

`ip route get 167.235.218.44 from 10.32.76.220` correctly returned
`via 10.32.64.1 dev phy1-sta0 table 102`.

Against an MPTCP echo server on the same host:

```text
MPJoinSynTx     0 -> 1
MPJoinSynAckRx  0 -> 1
SF_ESTABLISHED token=... remid=0 locid=2 saddr4=10.32.76.220 ... ifindex=hotel
```

So the hotel path **can** establish a second subflow on this hardware.

## Working transport: VLESS/BSBF on :16384

The working path is not the legacy Shadowsocks listener. `mptcp-hel1` runs a
BSBF **xray VLESS** inbound on **:16384** with:

```json
"streamSettings": { "sockopt": { "mark": 16384, "tcpMptcp": true } }
```

and a per-client VLESS UUID. A client profile was generated from that live
config (UUID not printed here). Xray refuses plain VLESS to a public IP, so the
controlled test used a private peer address with an OUTPUT DNAT:

```text
iptables -t nat -A OUTPUT -d 10.255.0.1 -p tcp --dport 16384 \
  -j DNAT --to-destination 167.235.218.44:16384
```

Client outbound: `vless` to `10.255.0.1:16384`, UUID, `sockopt.tcpMPTCP=true`.

With the same manual endpoints + metrics + per-source routing, a 4 MB
`speed.cloudflare.com` download produced:

```text
[ SF_ESTABLISHED] token=... locid=2 saddr4=10.32.76.220 daddr4=10.255.0.1 ... ifindex=hotel
[ SF_ESTABLISHED] token=... locid=2 saddr4=10.32.76.220 daddr4=10.255.0.1 ... ifindex=hotel
down HTTP=200 SIZE=4000000 SPEED=4709472   (~37.7 Mbit/s)
egress = 167.235.218.44
```

Per-interface RX deltas for that transfer:

| Interface | RX delta |
|---|---|
| c03rad0r `phy0-sta0` | ~159 KB |
| hotel `phy1-sta0` | **~4.13 MB** |

So the bond **worked** in the controlled test; the hotel path carried most of
the payload, with c03rad0r carrying the rest. The test script then restored the
production single-uplink state; the bond is **not persistent** yet.

## What did not work

The legacy Shadowsocks path (4 MB through `mptcp-hel1:65102`):

- egress was the server (`ifconfig.me` = 167.235.218.44);
- c03rad0r RX grew by ~4 MB;
- hotel RX grew by only ~1–2 KB;
- no `SF_ESTABLISHED` for the SS connection; only initial connections.

Why the SS path failed:

1. **The SS server is legacy `chacha20`.** Xray 26.10.21 rejects `chacha20`
   ("unknown cipher method"); a temporary AEAD listener (`chacha20-ietf-poly1305`)
   on `:65102` made xray work, but xray's `tcpMPTCP` did not produce a second
   subflow for the Shadowsocks outbound. The VLESS/BSBF path did.
2. **`bsbf-mptcp` helper is broken on ip-full 6.18.** It calls
   `ip mp e a ...` (an alias that fails on this iproute2), so its endpoints table
   stayed empty. Manual `ip mptcp endpoint add` works; the package needs
   `ip mptcp endpoint add`.
3. The kernel path manager needs **both main-table defaults with metrics**
   (10/20), not a single metric-0 default plus a high-metric backup. With that
   shape, the echo server and the VLESS path both got hotel subflows.

## Server facts

- `ss-server --mptcp` on `:65101` (chacha20)
- OMR on `:65500`
- BSBF xray on `:16384` (per-client configs; not reachable from the host path
  inspected due to container/deleted path)
- **`nft` not installed** → no server-side meter/enforcement
- Provider firewall allows 65101/65102/65500/65222; 443 is blocked externally

Temporary listeners (AEAD `:65102`, echo `:65102`) and their keys were removed
after the test. Production is back on the single c03rad0r uplink, bond disabled.

## Persistent rollout attempt (rc8, 2026-11-22)

The module path was then configured on production with:

- `upstream_wifi.bonded_sta_names=["hotel_sta"]` (PR #1261 guard);
- `mptcp_bonding` enabled, proxy-terminated, uplinks with static
  address/gateway, metrics 10/11, `subflow fullmesh`;
- a VLESS TPROXY client service and `10.255.0.1` DNAT.

It **did** apply the uplink bond: both endpoints (`subflow fullmesh`), main
defaults metric 10/11, per-source tables/rules, and a SOCKS VLESS transfer at
~4.3 MB/s. So the module path works for the uplink.

It was **not stable enough to leave enabled**:

1. **Hotel STA roamed to ch108 (DFS)** while radio1’s AP stayed ch36. The
   MT3000 AP+STA rule requires the same channel on one radio. Need channel
   follow/ACS on the AP, or a fixed BSSID/channel and an AP that matches.
2. **Interface rename during network restart** left an endpoint on a transient
   `if539` while c03rad0r was `phy0-sta0`. The module applied while the netdev
   was being renamed; a later reconcile needs the settled name/ifindex.
3. **DHCP reasserted c03rad0r metric 0** after the network restart, replacing
   the module’s metric-10 default. A route watcher/reapply is needed.
4. **Steering chain not loaded** in this run (`mptcp_steer` absent). The
   BusyBox-`paste` failure is fixed in rc8, but the apply result still needs a
   dedicated test with `fw4`/nft.
5. The xray TPROXY inbound (1082) and VLESS DNAT were running, but guest
   steering was never exercised end-to-end.

### Resolution: 5 GHz STA-only + hotel re-auth

The channel collision was removed by **disabling the 5 GHz APs** and dedicating
radio1 to the hotel STA. The 2.4 GHz APs stay on ch5 with the c03rad0r STA.
The hotel can roam channels without an AP on that radio to match.

After re-running the ASK4 guest login on the current BSSID, the persistent bond
worked:

```text
[ SF_ESTABLISHED] saddr4=10.32.76.220 daddr4=10.255.0.1 ... ifindex=hotel
[ SF_ESTABLISHED] saddr4=10.32.76.220 daddr4=10.255.0.1 ... ifindex=hotel
down HTTP=200 SIZE=4000000 SPEED=3136723   (~25 Mbit/s)
```

RX deltas: hotel ~3.08 MB, c03rad0r ~1.21 MB. A reboot then restored both
endpoints, metrics 10/11, the `mptcp_steer` chain, xray, and internet; the
post-reboot transfer fell back to c03rad0r until the hotel portal was
re-authenticated, then both paths carried payload again.

A local `tollgate-hotel-portal` watcher checks the hotel path every 5 minutes
and re-runs the ASK4 guest login when captive. This is a **local stopgap**; the
per-uplink portal client belongs upstream in #1253.

Current production: rc8, bond **enabled**, 5 GHz APs disabled, wallet 0 sats.

Remaining caveats:

- the client uses the server's **static per-client VLESS UUID**, not a paid
  per-session credential;
- the ASK4 guest form is first/last-name only and is replayed locally, not via
  the TollGate per-uplink portal code;
- the `mptcp_bonding` uplink addresses/gateways are static; a DHCP change needs
  a config update/reconcile;
- guest TPROXY steering is installed (`mptcp_steer` chain, TPROXY `:1082`,
  `fwmark 0x1`→table 100), but not yet exercised by a real guest. A synthetic
  veth-on-`br-tollgate` test was attempted and blocked because this OpenWrt
  image has no `veth` device type (`Error: Unknown device type`).

## Next changes needed

1. Productize the VLESS/BSBF transport: per-client VLESS UUID from the paid
   session, TLS or a private tunnel instead of the `10.255.0.1` + DNAT
   workaround, and a supervised client process.
2. Fix `bsbf-mptcp`'s helper to call `ip mptcp endpoint add` (not `ip mp`), or
   keep manual endpoints from the TollGate `mptcp_bonding` applier.
3. Keep both main-table defaults with metrics and the per-source tables.
4. Implement multi-STA (#1259) so the TollGate manager does not dismantle the
   second STA.
5. Add a boot/restart service for the manual bond (endpoints, routes, xray,
   DNAT) and verify reboot persistence.
6. Make the server a paid TollGate: kind-10021 ad with `capability=bonding`,
   Cashu session, per-client VLESS provisioning, nft meter/quota (not installed
   on the server yet).
7. Measure failover and the honest split; do not quote a multiplier.

## Honest status

- **The controlled production test bonded successfully**: both subflows
  established, hotel path carried ~4.13 MB of the 4 MB test, egress via the
  MPTCP server.
- **Production is not persistently bonding.** The test restored the single
  c03rad0r uplink; no bond service is enabled.
- The **ASK4 guest login** works, but no persistent ASK4 account was created;
  the email/room inputs were not used by the guest path.
- Do not quote a throughput multiplier; the MT6000 evidence remains ~1.2–1.3×
  best case, and this test was one short transfer.
