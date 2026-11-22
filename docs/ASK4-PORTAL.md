# ASK4 captive portal (Pendulum Hotel Guest)

Observed from `tollgate-76aa` on 2026-11-22 with a temporary STA; the original
uplink configuration was restored automatically after every probe.

## Network facts

| Item | Value |
|---|---|
| SSID | `Pendulum Hotel Guest` (open) |
| AP | B4:A2:5C:78:32:60, 2.4 GHz ch1, about -68 dBm |
| Lease | `10.32.77.42/20` |
| Gateway / DNS | `10.32.64.1` |
| Portal host | `gw.customer.ask4.lan` → `10.32.64.1` |
| Portal path | `/redirect.php` |
| Visible alternatives | `Pendulum VOIP`, `iQ Pendulum Hotel Staff` (WPA PSK) |
| 5 GHz | `Pendulum Hotel Guest` ch36/ch40/ch44, about -78 dBm |

## Two blockers the naive auto-login would hit

### 1. TollGate's NDS `ndsOUT` chain swallows the request

On the router:

```text
Chain ndsOUT (1 references)
RETURN  mark match 0x20000/0x30000
RETURN  mark match 0x30000/0x30000
DNAT    tcp dpt:80 to:192.168.99.1:2050
ACCEPT
```

So any **router-originated** HTTP request is DNATed to the local TollGate
portal, not the hotel portal. The NDS chain already trusts the portal marks;
the auto-login client must set `SO_MARK=0x20000` on its login socket to take
the `RETURN` branch. PR #1253 adds `portal_login_mark` for this.

### 2. The venue DNS is not the router resolver

`gw.customer.ask4.lan` resolves to `10.32.64.1` **when queried at 10.32.64.1**.
The router's dnsmasq returned `NXDOMAIN` because it was using the other
uplink's resolver. The login client must use `portal_login_dns=10.32.64.1`
(PR #1253) or an explicit IP URL.

## Recipe shape

```sh
tollgate config set upstream_wifi.portal_login_enabled=true
tollgate config set upstream_wifi.portal_login_mode=form   # or click_through
tollgate config set upstream_wifi.portal_login_url='http://gw.customer.ask4.lan/redirect.php'
tollgate config set upstream_wifi.portal_login_dns='10.32.64.1'
tollgate config set upstream_wifi.portal_login_mark=0x20000
# Only if the portal asks for a room/last name:
tollgate config set upstream_wifi.portal_login_username='<room-or-name>'
tollgate config set upstream_wifi.portal_login_extra='room=<room>&lastname=<name>'
```

The portal page itself was not captured: the hotel AP showed repeated
`CTRL-EVENT-BEACON-LOSS` and TCP/80 to the gateway did not answer during the
probe window. One stable capture is still needed to decide between
`click_through` and `form`, and to name the exact fields. The engine supports
both; the URL/DNS/mark above are measured.

## Operational note

The 2.4 GHz hotel AP is weak and beacons were lost during the probe. The 5 GHz
BSSID is about -78 dBm; positioning matters. Do not claim a stable bond until a
capture/transfer succeeds without beacon loss.
