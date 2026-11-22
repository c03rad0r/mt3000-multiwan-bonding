# Access and resume procedure

## Lab DUT

- Host: x280 (FIPS `x280.fips`, lab LAN `192.168.2.201`)
- Labgrid coordinator: `x280.fips:20408`
- Place: `gl-mt3000` (free at last check; acquire before use)
- DUT: `192.168.50.205` (management on the MT6000 LAN), serial FTDI
  `BG046H5F`

```sh
export LG_COORDINATOR=x280.fips:20408
labgrid-client places
labgrid-client -p gl-mt3000 acquire
labgrid-client -p gl-mt3000 ssh        # or plain ssh root@192.168.50.205
labgrid-client -p gl-mt3000 release
```

The MT3000 is only reachable through x280. If x280's FIPS link is down, wait
for it to recover; the device is not on the T14 LAN.

## Production device

- `tollgate-76aa.lan` / `192.168.2.1`
- Root login is password-based; the password is **not** stored in this repo.
- Wallet was 0 sats at inspection.

Before any change:

```sh
tar czf /tmp/etc-tollgate.tgz /etc/tollgate
tar czf /tmp/etc-config.tgz /etc/config
tollgate version > /tmp/version.txt
apk list -I > /tmp/packages.txt
sha256sum /tmp/etc-*.tgz
```

Pull the backup off-device with `scp -O` (OpenWrt has no sftp-server) or
`ssh ... 'tar czf - ...' > backup.tgz`.

## 2026-11-22 blocker

At about 00:30 the x280 FIPS link and the MT3000 behind it stopped answering.
`tollgate-76aa` still answers on 192.168.2.1 but cannot reach the MT6000
(192.168.2.2) or the MT3000. No device or code change caused it.

Resume order:

1. Confirm `x280` is reachable (`ssh x280-direct hostname`).
2. Acquire `gl-mt3000`; confirm `192.168.50.205` answers.
3. Read `/root/lab-uplinks.sh` (written before the outage) and verify the two
   STA sections associated, got `wwan`/`wwan2` addresses, and have internet.
   The script stops `/etc/init.d/tollgate-wrt` first so the single-STA manager
   cannot dismantle the manual pair; restart it before product-path testing.
4. Install the PR-branch build, then run the MPTCP proof.
5. Only after that, plan the `tollgate-76aa` upgrade.

## Current production state (2026-11-22)

- `tollgate-76aa`: OpenWrt 25.12.5, TollGate **0.6.0-rc6** (commit `a4c2462b`),
  wallet 0 sats.
- Uplink: `c03rad0r` on radio0, **`sae-mixed`** (WPA2 `psk2` gave
  `WRONG_KEY`), address `10.110.131.142`, internet verified.
- Bonding: **disabled**. Single-uplink state restored after the hotel dry run.
- ASK4: `gw.customer.ask4.lan:3145/redirect.php`; auto-login disabled until the
  guest/existing-account flow is known.
- Rollback: `/root/rollback/` on the device, `/var/tmp/tg76aa-backup/` on T14.

## Known-good artifacts

| Artifact | Path / hash |
|---|---|
| MT3000 backup (on x280) | `/var/tmp/mt3000-backup-20261119-080802` |
| `etc-tollgate.tgz` sha256 | `d184f65d0b92b2429acf2a930aab7f5a426ca7ca62677f916b259d73bbc84873` |
| `etc-config.tgz` sha256 | `2942f35aa10ea5f7fe6b20c77e18b6bf48495d4e3bc6fbe8f6fea6da98928c2c` |
| Main-only test APK | `tollgate-wrt-0.6.0_rc4-r0.apk` sha256 `15e41df24b2ffe552c449477907ab91f46ad93b694149006df179df7bfd88fdd` |
| Integration APK (all seven PRs + ASK4 fix) | `tollgate-wrt-0.6.0_rc6-r0.apk` sha256 `86c2a1736263c3f654fcc2e8b20de347fd6b6782bce5d812a7cfb04e47d19414` |
| Firmware image | `openwrt-25.12.6-…glinet_gl-mt3000-squashfs-sysupgrade.bin` sha256 `c2965b9d49d2ff42bf00c4f04ef73eaae904f553c68af47332fec702b30bf6f4` |
