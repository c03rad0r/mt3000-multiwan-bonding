# Scripts

Planned/verification helpers. None have been run against the final bonded
setup yet.

- `verify-bond.sh` — `ip mptcp endpoint show`, limits, `ip mptcp monitor`,
  `ss -Mti` for `tcp-ulp-mptcp` and `subflows:2`.
- `verify-steering.sh` — confirm pre-auth traffic is **not** in the chain and
  authenticated traffic is, using nft counters.
- `measure.sh` — per-path vs bonded `iperf3` with explicit source binds.
- `failover.sh` — kill the dominant path mid-transfer and assert survival.

Every script must be read-only unless its name says apply/teardown, and must
refuse to run without a backup marker on the production device.
