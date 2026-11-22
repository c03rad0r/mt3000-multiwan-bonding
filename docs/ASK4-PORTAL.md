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
| Portal port | **3145** (`http://gw.customer.ask4.lan:3145`) |
| Redirect path | `/redirect.php` |
| Redirect page | GET form, no named fields, “Continue online” → portal root |
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
tollgate config set upstream_wifi.portal_login_mode=click_through
tollgate config set upstream_wifi.portal_login_url='http://gw.customer.ask4.lan:3145/redirect.php'
tollgate config set upstream_wifi.portal_login_dns='10.32.64.1'
tollgate config set upstream_wifi.portal_login_mark=0x20000
```

The redirect page is a `GET` form with **no named fields** (“Continue online”).
PR #1253 now accepts that shape and submits the action with an empty query.

The sign-up page has three paths:

| Path | Result |
|---|---|
| Create Account `/en/speed` | HTTP 500 in the probe |
| Already Signed Up `/en/existing-customer` | redirects to `https://myaccount.ask4.com/?maccheck=1&mac=<sta-mac>` |
| Guest Login `/en/guest-signup` | **200**, 65 KB form; this is the working path |

The guest form asks for **first name + surname + CSRF token** only. Posting it
redirects to `https://pendulumhotel.co.uk/`, and the same STA MAC then has
internet (`ifconfig.me` = 31.205.35.59). **No email, room number, password, or
email verification was used.** A persistent ASK4 account was not created; this
is a guest/MAC session.

A per-uplink HTTP client and a session cookie from the hotel path are still
required for full automation (the signup host rejects non-hotel clients with
`/en/wrong-network`).

## What was captured

- `/redirect.php` (port 3145) → 1369-byte page with the no-field GET form.
- Submitting the action `GET /` returned the 99542-byte `ASK4 - Sign up` page
  (or `302 Location: https://signup.ask4.com` depending on the probe).
- `https://signup.ask4.com` is the real signup host. Fetched from a non-hotel
  client it redirects to `/en/wrong-network`; it only serves the form when the
  request originates from the hotel uplink.
- `/en/guest-signup` with the hotel session returned the 65 KB first/last-name
  form; the POST authorized the MAC and redirected to the hotel site.
- `/en/speed` returned HTTP 500; `/en/existing-customer` went to
  `myaccount.ask4.com` with `maccheck=1`.

## Consequence for the auto-login engine

The ASK4 flow is **not** a same-host click-through:

1. It redirects to an external host (`signup.ask4.com`).
2. That host validates the client's hotel-side public IP, so the HTTPS request
   must leave through the **hotel STA**, not the default uplink.
3. It needs the guest/existing-account form, which is only served to the hotel
   client.

PR #1253's current same-host guard and default-route HTTP client are therefore
not sufficient for ASK4. The next change needs:
- a configured allow-list for external portal hosts (never arbitrary), and
- a per-uplink HTTP client (bind/route through the target STA) for both the
  portal page and the signup site.

The working guest recipe is simple once bound to the hotel STA: GET the
redirect page, submit the POST to `/en/guest-signup` with configured first and
last name, then re-probe. This is why production auto-login is disabled and the
bond is not persistently enabled yet.

## Operational note

The 2.4 GHz hotel AP is weak and beacons were lost during the probe. The 5 GHz
BSSID is about -78 dBm; positioning matters. Do not claim a stable bond until a
capture/transfer succeeds without beacon loss.
