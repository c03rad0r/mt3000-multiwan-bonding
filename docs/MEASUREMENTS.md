# Measurements

No bonded throughput or failover numbers yet. The lab became unreachable after
the device was upgraded and the two-STA configuration was written, before
association/DHCP and the MPTCP proof could be verified.

When the lab returns, record:

1. Per-path baseline with `iperf3` bound to each uplink address.
2. Bonded run with the same server/port and both endpoints registered.
3. `ss -Mti dst <server>` showing `tcp-ulp-mptcp` and `subflows:2`.
4. Byte deltas per path (the 91/9-style split from the MT6000 work).
5. Mid-transfer failover: kill the dominant path, confirm the connection
   survives and the remaining path carries payload.
6. MT3000 CPU/airtime cost with both AP+STA radios active.

Do not report a number as "2×" or "faster" without the per-path baseline; the
MT6000 evidence is approximately 1.2–1.3× best case, often a lower median.
