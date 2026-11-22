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

## What did not work

A 4 MB xray/Shadowsocks transfer through `mptcp-hel1:65102`:

- egress was the server (`ifconfig.me` = 167.235.218.44);
- c03rad0r RX grew by ~4 MB;
- hotel RX grew by only ~1–2 KB;
- no `SF_ESTABLISHED` for the SS connection; only initial connections.

Likely causes, in order:

1. **The SS server is legacy `chacha20`.** Xray 26.10.21 rejects `chacha20`
   ("unknown cipher method"); a temporary AEAD listener (`chacha20-ietf-poly1305`)
   on `:65102` made xray work, but xray's `tcpMPTCP` still did not produce a
   second subflow for the Shadowsocks outbound.
2. **`bsbf-mptcp` helper is broken on ip-full 6.18.** It calls
   `ip mp e a ...` (an alias that fails on this iproute2), so its endpoints table
   stayed empty. Manual `ip mptcp endpoint add` works; the package needs
   `ip mptcp endpoint add`.
3. The kernel path manager needs **both main-table defaults with metrics**
   (10/20), not a single metric-0 default plus a high-metric backup. With that
   shape, the echo server got a hotel subflow.

## Server facts

- `ss-server --mptcp` on `:65101` (chacha20)
- OMR on `:65500`
- BSBF xray on `:16384` (per-client configs; not reachable from the host path
  inspected due to container/deleted path)
- **`nft` not installed** → no server-side meter/enforcement
- Provider firewall allows 65101/65102/65500/65222; 443 is blocked externally

Temporary listeners (AEAD `:65102`, echo `:65102`) and their keys were removed
after the test. Production is back on the single c03rad0r uplink, bond disabled.

## Next changes needed

1. Use a client that actually sets MPTCP for the tunnel (ss-local with
   `--mptcp`, a patched SS client, or xray VLESS/BSBF), or switch the server to
   an AEAD SS/VLESS transport that xray can use.
2. Fix `bsbf-mptcp`'s helper to call `ip mptcp endpoint add` (not `ip mp`), or
   keep manual endpoints from the TollGate `mptcp_bonding` applier.
3. Keep both main-table defaults with metrics and the per-source tables.
4. Implement multi-STA (#1259) so the manager does not dismantle the second STA.
5. Then test full payload/failover and only then enable steering/production.

## Honest status

- **Not currently bonding.** No production bond is enabled.
- The **mechanism** is proven (hotel `SF_ESTABLISHED`).
- The **full guest payload bond** is not yet proven.
- Do not quote a throughput multiplier; the MT6000 evidence remains ~1.2–1.3×
  best case.
