# Topology and radio constraints

## Target

```text
mobile hotspot (2.4 GHz) ─┐
                          ├─ GL-MT3000 (TollGate) ─ guest portal
hotel guest (5 GHz) ──────┘
```

Observed from `tollgate-76aa` (GL-MT3000 class):

| Role | Band | Channel | Signal | Encryption |
|---|---|---|---|---|
| Mobile hotspot | 2.4 GHz | 5 | about -42 dBm | WPA2/WPA3 transition |
| Hotel guest | 5 GHz | 36 | about -78 dBm | open |
| Hotel guest (backup) | 2.4 GHz | 1 | about -71 dBm | open |

## Why the channel plan is forced

`iw phy` on the MT3000 reports one AP+STA channel per radio:

```text
#{ AP, mesh point } <= 16, #{ managed } <= 19, total <= 19, #channels <= 1,
STA/AP BI must match
```

So a radio cannot run an AP on ch1 and an STA on ch5. The layout must be:

```text
radio0: APs ch5   + STA hotspot ch5
radio1: APs ch36  + STA hotel ch36
```

Moving radio0's APs from ch1 to ch5 will briefly bounce 2.4 GHz clients. The
5 GHz AP is already ch36, which matches the hotel guest's strongest BSSID.

## Address plan on `tollgate-76aa`

| Interface | Address | Purpose |
|---|---|---|
| `br-private` | 192.168.2.1 | management / private AP |
| `br-lan` | 192.168.1.1 | private LAN AP |
| `br-tollgate` | 192.168.99.1 | guest portal bridge |
| `phy0-sta0` | DHCP | hotspot uplink (2.4 GHz) |
| `phy1-sta0` | DHCP | hotel uplink (5 GHz) |

The bond treats `phy0-sta0` and `phy1-sta0` as its two uplinks; portal and
management traffic must never be steered into the bond (`policy: guest_only`).

## Performance expectation

The MT6000 work measured approximately **1.2–1.3× best case**, with a median
that can be lower than the best single path. Two subflows converge at one
remote host and share its bottleneck/RTT. The hotel path is weak (about -78
dBm) and AP+STA shares airtime; treat this as resilience and session
continuity, not a bandwidth multiplier. If both APs share one upstream ISP,
there is no throughput gain at all.
