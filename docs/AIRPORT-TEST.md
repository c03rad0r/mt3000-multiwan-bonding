# Manchester Airport captive-portal test (2026-11-22)

Device: `tollgate-76aa`, OpenWrt 25.12.5, TollGate **0.6.0-rc9** (`eb879697`).
Uplink: `_FreeWifi` (open, 5 GHz, BSSID `08:18:14:B2:9A:65`).
Address: `10.250.73.81/23`; gateway `10.250.73.254`; DHCP DNS
`103.247.36.36`, `103.247.37.37`.

The airport hands out a different /23 per AP: the first association was
`10.250.77.84/23` on `08:18:14:AE:7C:A5`; after a roam the current lease is
`10.250.73.81/23` on `08:18:14:B2:9A:65`. The bond config must be updated (or
reconciled) when the lease changes.

## Portal flow

The airport uses a **Purple** portal behind an Aruba controller. It is not a
single form:

1. HTTP is intercepted; the first response is a meta-refresh to the Aruba
   redirect.
2. The controller redirects to `purpleportal.net` with the client MAC, IP, AP
   name/group and switch IP in the query.
3. The portal offers two plans:
   - **MAG MAN FREE Wi-Fi** — 2 hours, 2 GB, 7.5 Mbps, free;
   - MAG MAN PREMIUM Wi-Fi — 24 hours, 15 GB, 30 Mbps, £2.
4. Registration/sign-in is by **email** plus a terms checkbox and a CSRF token.
   A one-time email address is not required; a normal address is accepted.
5. After the free plan is selected, the portal validates the session and
   redirects to the controller's `cmd=authenticate` URL. Following that URL
   returns the “External Welcome Page” and authorizes the STA MAC.

This was completed **once**, manually, with the operator's own catch-all
address. No additional accounts, no MAC change and no node-key change were
made. The session is tied to the STA MAC `96:83:c4:8c:59:c2`.

The current per-uplink portal client handles a **single form**
(`click_through` or `form`). It cannot drive Purple's
registration → plan → controller-token sequence, so it was not enabled for this
uplink. A multi-step recipe engine is the next portal feature (see below).

## Result

- Airport uplink verified: `https://ifconfig.me/` returned `62.233.55.124`.
- The default route is `10.250.73.254` on `phy1-sta0` (metric 10).
- The device serves its APs while using this uplink.
- The free session is **2 hours / 2 GB**; it is not re-created by rotating
  identity.

### Re-auth after a roam

After the STA roamed to a new BSSID the portal still showed the account's
active free plan (“2 GB of 2 GB”, time remaining), but the new client was
walled. The page's **Continue online** action is `GET /access/validate`; that
returns the controller's `cmd=authenticate` URL. Following it (and its
meta-refresh) authorized the new MAC/IP. This was the **same account and the
same device**, not a new identity.

## Bond blocker on this network

The MPTCP server `167.235.218.44` is **unreachable from the airport network on
every tested port** (22, 80, 443, 16384, 65101, 65102, 65500, 65222): every
connection timed out. T14's own probe through the same airport uplink failed
identically, so this is the network's egress policy, not the server.

Consequences:

- The two-path VLESS bond cannot form here: the second subflow has no far end.
- The earlier hotel proof (both `SF_ESTABLISHED`, ~4.7 MB/s, hotel RX ~4.13 MB)
  remains the working bond evidence; this airport network is not a valid
  second-path test site.
- The device was left in a stable **single-uplink** state: `mptcp_bonding`
  disabled, the VLESS service disabled, the airport default route installed.

## What this changes in the plan

1. **Portal recipes need steps, not one form.** Purple is
   redirect → email registration → plan selection → controller
   `cmd=authenticate`. A recipe schema with ordered steps, a cookie jar and an
   allow-list for `purpleportal.net` plus the controller is required.
2. **Do not treat “portal authorized” as “MPTCP server reachable.”** The
   airport walled garden allowed the portal and public web, but blocked the
   bond server. A bond member needs an explicit far-end reachability check
   before it is added.
3. **No identity rotation.** The free session is one authorized session. When
   it expires, the correct behaviour is to drop the uplink and fall back, or use
   the airport's paid plan; not to rotate a MAC or the node key.
