# Ansible (skeleton)

Not yet run. The target is a GL-MT3000 running OpenWrt 25.12.x with TollGate
from a `main`/PR build.

Planned roles, adapted from `felixfelix-bot/mt6000-mptcp-bonding`:

- `common` — `ip-full`, `xray-core`, `kmod-wireguard`, `wireguard-tools`,
  `kmod-nft-tproxy`; apk-aware.
- `uplinks` — two STA sections, distinct `wwan`/`wwan2`, AP channel matching,
  and a guard against the single-STA manager.
- `bond` — `proxy-terminated` endpoints and per-uplink policy routing.
- `proxy` — xray with `tcpMPTCP: true`, loopback SOCKS + TPROXY.
- `steering` — NDS-aware fw4 chain; no pre-auth steering.
- `teardown` — remove endpoints, rules, tables, chain, and restore backup.

Inventory should use the labgrid place (`gl-mt3000`) or the production device,
never hardcoded credentials. All PSKs/passwords come from environment/Ansible
vars, never git.
