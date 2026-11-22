# Upstream gaps in `tollgate-module-basic-go`

Repository: <https://github.com/OpenTollGate/tollgate-module-basic-go>
Baseline inspected: `main` @ `959280db` (v0.6.0-rc4).

| Gap | Code reference | Consequence on MT3000 | PR / issue |
|---|---|---|---|
| Schema/bond policy mismatch | `src/config_manager/config_schema.go` advertises `always`; `src/mptcp_bonding/config.go` validates `all` | `policy=all` cannot be set; `policy=always` fails at bond load | #1257 |
| `ip-full` not a dependency | `packaging/Makefile` DEPENDS; runbook said `opkg` only | BusyBox `ip` has no `mptcp`; bond apply fails on a stock image | #1256 |
| `transport`/`tunnels` not configurable | `MPTCPBondingConfig` and schema lacked both; `main.bondConfigJSON` marshals the typed struct | NAT-safe `proxy-terminated` shape unreachable through the CLI | #1255 |
| No non-tunnel policy routing | `mptcp_bonding.EndpointPlan` only adds endpoints; `TunnelRoutingScript` covers tunnels only | Second subflow can follow the primary default route and be dropped | #1254 |
| No guest steering | `ShouldSteer` had no non-test consumer; #972/#974 call steering future work | Bond exists but guest traffic never enters it | #1258 |
| Naive steering would bypass payment | NDS marks are in `packaging/files/etc/nftables.d/20-nds-enforce.nft`; a CIDR-only rule ignores them | Pre-auth guest traffic would be a free tunnel | #1258 |
| No captive-portal auto-login | `src/wireless_gateway_manager/portal_login.go` only opens a 2-minute human window | Unattended router cannot pass a hotel portal; bond never forms | #1253 |
| NDS `ndsOUT` hides the venue portal | `iptables -t nat -L ndsOUT`: router-originated TCP/80 is DNATed to `192.168.99.1:2050` | Login request hits TollGate's own portal, never ASK4. Fixed by `SO_MARK=0x20000` in #1253. | #1253 |
| Venue DNS not used | Router dnsmasq returned NXDOMAIN for `gw.customer.ask4.lan`; `10.32.64.1` resolves it | Portal host cannot be found on a multi-uplink box. Fixed by `portal_login_dns` in #1253. | #1253 |
| Portal redirects to an external host | ASK4 redirects to `https://signup.ask4.com`, which serves `/en/wrong-network` unless the request comes from the hotel uplink | The same-host guard and default-route client cannot complete the flow. Needs a host allow-list + per-uplink HTTP client. | follow-up to #1253 |
| No per-uplink portal login | The manager's portal path probes only the single active STA | A hotel STA that is not the default route is never logged in; bond uses a captive path. | #1259 + follow-up |
| No MPTCP-capable SS client path | xray rejects the server’s legacy `chacha20`; AEAD xray `tcpMPTCP` did not produce a second subflow for Shadowsocks | Legacy SS is not the bond transport. The working path is the server’s **VLESS/BSBF xray :16384** with `tcpMptcp`; per-client UUID provisioning is static today. | paid-server phase |
| No paid VLESS session | Server advertises nothing; UUID configs are static files; no Cashu session or per-client credential issuance | A paid TollGate server needs kind-10021 ad + kind-1022 session + per-client VLESS UUID. | paid-server phase |
| `bsbf-mptcp` helper alias | Helper runs `ip mp e a ...`, which fails on ip-full 6.18 | Endpoint table stays empty; manual `ip mptcp endpoint add` works. | external package fix |
| Server has no meter/enforcement | `nft` not installed on `mptcp-hel1` | Paid-bond design cannot meter/enforce until nft counters/quota/tc are deployed. | paid-server phase |
| AP+STA channel follow | Hotel STA roamed to ch108; radio1 AP stayed ch36 | **Worked around** by disabling the 5 GHz APs and dedicating radio1 to the STA. Upstream channel-follow/ACS would restore the 5 GHz AP. | follow-up |
| Endpoint identity across netdev rename | Module applied while c03rad0r was transient `if539`; endpoint stayed on the old name | Did not recur on a clean reboot (both endpoints on `phy0-sta0`/`phy1-sta0`). Still needs a reconciler for network restarts. | follow-up |
| DHCP metric reassert | c03rad0r default returned to metric 0 after an install-time network restart | Did not recur on a clean reboot (metrics 10/11 persisted). A route watcher is still advisable. | follow-up |
| Steering apply not verified on fw4 | rc8 fixed `paste`; the manual chain loads; the module’s persistent chain is present after reboot | Guest traffic still needs an end-to-end test. | #1258 follow-up |
| Single-STA manager | `Connector.Connect` calls `disableOtherSTAInterfaces`; `CleanupStaleSTAs` disables non-active STAs; both `tollgate_sta_2g/5g` use `network=wwan` | Two simultaneous uplinks are dismantled; no product-path 2.4+5 bond | #1259 |
| Released build predates MPTCP | tag `v0.6.0` has no `src/mptcp_bonding`; `main` is 177 commits ahead | The installed v0.6.0 cannot bond at all | n/a — use a main build |
| Kernel CVE floor | #1159: fix starts at 6.12.113; 25.12.5=6.12.94, 25.12.6=6.12.112 | MPTCP listener exposure on an untrusted hotel network | production decision |

## What is already merged on `main`

- `src/mptcp_bonding`: config validation, endpoint plan, limits, idempotent
  apply, disable flush, interface-event reconciler.
- `proxy-terminated` transport (#1009).
- Per-uplink WireGuard tunnel renderers (plan/routing/firewall).
- Multi-WAN objectives + mwan3 applier (#1124/#1126/#1127) — a different,
  per-flow engine, not MPTCP aggregation.
- Disjointness preflight and counter scripts.
- Boot wiring through `applyBondConfig`.

## What is still needed for the product path

1. Merge/rebase #1254, #1255, #1256, #1257, #1258, #1253.
2. Implement #1259 (multi-STA) or accept a manual two-STA configuration that
   the manager does not dismantle.
3. Add `kmod-nft-tproxy` as a runtime prerequisite for steering.
4. Add an xray/proxy lifecycle (or document the operator-supplied proxy).
5. Add the MT3000 hardware/CVE runbook (#H, not yet filed).
