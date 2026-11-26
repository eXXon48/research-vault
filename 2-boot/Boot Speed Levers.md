---
tags: [cmu150, boot, sm, timing, performance]
---

# Boot Speed Levers

Changes that were tested to make the CMU150 boot faster, with the measured effect of each.
Milestones and the stock dependency chains are in [[Boot Timeline]].

## Overview

Most of the boot is the System Manager phase: services start in series behind `jciMMUI`, the
stage barriers and the Bluetooth chain, with both cores saturated. The changes that help move a
dependency or remove repeated work: start `mfid` on `stage_3`, keep the GStreamer registry across
boots, lower the log level. Moving load between cores trades the SM finish time against the Home
screen. Moving files to faster storage makes the boot slower.

To show in five boots, a lever has to move a milestone by more than the boot-to-boot spread:
under 1 s up to `stage_3`, about 2 s after it ([[Boot Timeline]], "Variation between boots").

Related: [[HMI Performance Levers]], [[FlashFX Read Path]], [[SM Service Graph]].

## Summary

| Change | Effect | Boots |
|---|---|---|
| `mfid` depends on `stage_3` instead of `jciBCA` | Android Auto host side ready ~6–7 s sooner; Home unchanged | 6 |
| GStreamer registry on `/data_persist` (`GST_REGISTRY`) | Android Auto projection 38.5 s average, against 42–55 s | 3 |
| `COMMON_LOG_LEVEL=Warn` | `jciCARPLAY` 2.3 s and `RUNNING` 3.1 s earlier; boot timing markers disappear | 1 |
| CPU-heavy services moved from cpu0 to cpu1 | `bds` ~2.8 s, `jciCARPLAY` ~1.3 s, `RUNNING` ~3 s earlier; **Home 4–12 s later** | 3 + 8 |
| Disclaimer `Timeout` sent at once (GUI) | Home 0.2–0.9 s after `DisposeIntroVideo` | 5 |
| `stage_vr` sleep removed | `RUNNING` ~1.5 s earlier, inside the spread | 3 |
| `ffxread01` pinned to cpu1 | no measurable gain | 3 |
| `jcidvd`, `jcitv` set `autorun="no"` | no reliable effect on Home | 6 |
| hot rootfs files served from eMMC | **slower**: Android Auto ready 29.9 s after `stage_1`, against 23.45 s | 1 |

## Reference

### `mfid` on `stage_3`

`mfid`'s `jciBCA` dependency was changed to `stage_3` in both SM files. `mfid` uses neither
Bluetooth nor D-Bus, so its `jciBCA` dependency is ordering only ([[Boot Timeline]], "Critical
chain to Android Auto"). b51–b53 also had `jcidvd` and `jcitv` set `autorun="no"`. GUI tweaks
applied, wireless Android Auto USB adapter. Seconds since kernel start:

| | `mfid` | `devicemanager` | `jciAAPA` | AAP session | Home | `AAP_PROJECTION_SETUP` |
|---|---|---|---|---|---|---|
| stock dependencies (b46, b47) | 34.4–34.6 | 38.5–39.4 | 41.5–41.7 | 42.3–42.5 | 32.4–33.5 | 54.4–57.7 |
| `mfid` → `stage_3` (b48–b50) | 24.8–25.4 | 30.7–34.0 | 34.1–37.0 | 35.0–37.6 | 33.3–34.6 | 57.8 / 46.4 / 42.2 |
| + DVD/TV off (b51–b53) | 24.6–25.3 | 31.7–32.8 | 34.7–35.8 | 35.6–36.1 | 33.2–34.7 | 55.2 / 55.6 / 42.1 |

- `jciAAPA` and the AAP session are ready ~6–7 s sooner. Home does not move.
- `jciAAPA` also needs `jciRM` (~25.5 s) and `jciUpdatea` (~28.8 s), so even with
  `devicemanager` up early it cannot start before ~29 s.
- With a wireless adapter, the adapter sets the projection time, not the head unit
  ([[Boot Timeline]], "Wireless Android Auto adapter"). A phone on a USB cable was not tested.

### GStreamer registry

- GStreamer 0.10 on glib 2.22 takes its home directory from the passwd entry, not from `$HOME`
  (the `HOME=/` of SM services is ignored). That home is `/root`, a stock symlink to `/tmp/root`
  (tmpfs, created by `init_cmu`).
- So each boot writes the registry to `/tmp/root/.gstreamer-0.10/registry.arm.bin` (~66 KB) and
  loses it at the next boot. `gst-plugin-scanner` rescans every plugin at ~40 s, during the
  Android Auto session setup. It runs on every stock boot (two scanner processes in one boot).
- `GST_REGISTRY` (honoured by libgstreamer-0.10) sets the registry file path. SM services inherit
  their environment from `/etc/profile`, a regular rootfs file.
- With `export GST_REGISTRY=<file on /data_persist>` in `/etc/profile`, the scanner ran only on
  the first boot. Android Auto projection with a wireless adapter: 37.7 / 39.4 / 38.5 s uptime,
  against 42–55 s with the scanner.

### Log level

- The application log level is set in `/config/common/log/settings.txt` (rootfs):
  `COMMON_LOG_FORMAT=SYSLOGNGEXT`, `COMMON_LOG_STRIP=True`, `COMMON_LOG_LEVEL=Info`.
  `libjcicommon.so` reads it at each process start. There is no per-zone level key.
- `jci-log.sh level <level>` writes that file. `jci-log.sh console disable` and
  `jci-log.sh filter` also exist.
- `COMMON_LOG_LEVEL=Warn` cut one boot's log from about 20 000 lines to about 1 700 (to uptime
  ~260 s). In that boot, `jciCARPLAY` started 2.3 s and `SM_SERVER_CMD_RUNNING` 3.1 s earlier than
  in the Info boot before it (one boot each).
- Cost: the usual boot marks are Info lines and are missing at Warn (`HomeScreen` ctxtChg,
  `[Androidauto]Available Status true`, `USB_CARPLAY_1] Initialized OK`).
- Log volume and destinations: [[Boot Timeline]], "Boot log"; [[Syslog-ng]].

### Moving load from cpu0 to cpu1

The `<services>` default in `sm.conf` is `affinity_mask="0x01"`, so services run on core 0 unless
they set `0x02` ([[HMI Performance Levers]], "System Manager: core affinity"). Core 0 does
roughly six times the work of core 1.

**`ffxread01` only.** `taskset -p 2 <ffxread01>` at 5.9 s, with the `/proc` sampler running (one
boot shown, three run):

- `ffxread01` runs on cpu1 only (4.68 cpu-s up to `stage_3`).
- cpu0 busy drops to 31–78 % between 8 s and 24 s; cpu1 rises to 56–86 %.
- From 26 s to 68 s cpu0 is still saturated.

| Milestone | pinned | unpinned |
|---|---|---|
| `stage_3` | 24.63 | 25.24 |
| `bds` | 33.26 | 33.92 |
| `devicemanager` | 40.42 | 42.51 |
| `jciAAPA` | 44.34 | 45.43 |
| `jciCARPLAY` | 46.86 | 47.31 |
| `RUNNING` | 59.58 | 59.72 |

Three pinned boots: `stage_3` 24.63 / 24.51 / 25.60, `bds` 33.26 / 33.36 / 33.80, `RUNNING`
59.58 / 59.40 / 59.08. That is no measurable gain over unpinned boots without the sampler
(`stage_3` 24.09–25.08, `RUNNING` 57.90–59.82). Relieving cpu0 before `stage_3` does not shorten
the boot; after `stage_3`, SM services saturate cpu0.

**Services.** Every thread of `natp`, `L_jciVBS`, `L_jciCARPLAY`, `L_jciMMUI`, `L_settings`,
`L_jciblmVdt`, `L_jciVwm`, `vim_app`, `syslog-ng` and `dbus-daemon` moved to cpu1
(`taskset -p 2`) on first sight:

With the sampler (2.4 s scan):

| boot | `stage_3` | `bds` | `devicemanager` | `jciAAPA` | `jciCARPLAY` | `RUNNING` |
|---|---|---|---|---|---|---|
| 1 | 24.65 | 31.52 | 40.59 | 43.31 | 46.07 | 56.96 |
| 2 | 24.50 | 31.36 | 41.80 | 44.19 | 46.46 | 57.58 |
| 3 | 23.74 | 30.30 | 39.77 | 42.49 | 45.43 | 56.47 |

Without profiling (a scan of `/proc/*/stat` every 2 s):

| boot | `stage_3` | `bds` | `devicemanager` | `jciAAPA` | `jciCARPLAY` | `RUNNING` |
|---|---|---|---|---|---|---|
| 1 | 23.60 | 30.22 | 40.04 | 42.72 | 44.58 | 55.56 |
| 2 | 24.51 | 31.85 | 42.25 | 45.26 | 46.28 | 57.09 |
| 3 | 23.75 | 30.39 | 40.74 | 43.65 | 44.93 | 55.70 |

- Against six unmoved boots (medians `bds` 33.16, `jciCARPLAY` 46.27, `RUNNING` 58.73): `bds`
  ~2.8 s earlier, `jciCARPLAY` ~1.3 s, `RUNNING` ~3 s. `stage_3`, `devicemanager` and `jciAAPA`
  stay within the spread. The chain from `bds` to `jciAAPA` is not CPU-bound: it absorbs the
  earlier `bds` start.
- With the move, cpu1 becomes the saturated core at 27–48 s (92–100 %), and with `natp` on it
  also at 57–66 s, while cpu0 drops to 8–25 %.
- `dbus-daemon` (both instances) and the main thread of `vim_app` come back to cpu0 after being
  moved at ~5.8 s; their other threads stay on cpu1. No stock script pins them with `taskset`,
  so something else resets the affinity (not identified).

**Effect on the screens.** `ctxtChg` times (uptime, s) from `running_log`:

| Boots | `IntroAnimation` | `Disclaimer` | `HomeScreen` |
|---|---|---|---|
| unmoved (5 boots: plain, sampler, `ffxread01` pinned) | 24.9–26.9 | 30.4–32.1 | 41.0–41.9 |
| services moved to cpu1 (8 boots) | 23.9–25.6 | 29.6–31.1 | 46.0–53.4 |

- The move brings `RUNNING` ~3 s earlier but delays `HomeScreen` by 4–12 s.
- `opera` already runs on cpu1 (6.8 CPU-s there in the first 62 s). `jciMMUI`, `jciVwm`,
  `syslog-ng`, `natp` and the others moved next to it compete with it while the Disclaimer timer
  (3500 ms in `systemApp.js`) runs.
- Unmoved, the Disclaimer still lasts 9.3–10.0 s against its 3.5 s timer.

### Disclaimer timer

The Disclaimer's displayed callback sends `DisposeIntroVideo` and starts the 3500 ms timer
([[Boot Timeline]], "Screens during boot"). Sending `Timeout` in the same callback, with no timer,
gives `HomeScreen` 0.2–0.9 s after `DisposeIntroVideo` (five boots).

### `stage_vr` sleep

`stage_vr.sh` is `sleep "$1"` then `touch /var/run/stage_vr_ready` (`args="5s"`). The test killed
the `sleep 5s` child as soon as it appeared (`pgrep -f stage_vr.sh`, `pkill -P`); the script then
creates the ready file itself. No configuration changed. Both runs had the services moved to cpu1.

| boot | `jciCARPLAY` | `stage_vr` | `natp` | `jciTTSPlayer` start → started | `RUNNING` | `jciCARPLAY` → `RUNNING` |
|---|---|---|---|---|---|---|
| wait cut 1 | 46.38 | 47.19 | 47.23 | — | 54.72 | 8.3 |
| wait cut 2 | 46.74 | 47.51 | 47.56 | 47.59 → 52.18 (4.6) | 54.35 | 7.6 |
| wait cut 3 | 43.25 | 44.72 | 44.76 | 44.78 → 50.74 (6.0) | 53.45 | 10.2 |
| move only | 43.33 | 48.55 | 48.58 | 48.60 → 52.09 (3.5) | 54.22 | 10.9 |

- With the move only, `jciCARPLAY` → `RUNNING` is 10.8–10.9 s on all four boots; with the wait
  cut it is 7.6–10.2 s.
- `natp` reports started within 0.05 s, but `jciTTSPlayer` (depends on `natp`,
  `ping_timeout=30000`) then takes 4.6–6.0 s instead of ~3.5 s. Part of the removed wait comes back
  as `natp` initialisation.
- Net effect on `RUNNING`: about 1.5 s (mean 54.2 s against 55.6 s), within the boot-to-boot
  spread of the earlier chain (`devicemanager` 4–8 s).
- No board reset; UI up on all three boots. `natp`, `jciTTSPlayer` and `jciVUI` carry
  `reset_board="yes"`.

### DVD and TV services off

`jcidvd` and `jcitv` have no dependents in the SM graph. Set `autorun="no"` in both SM files, on a
unit without DVD/TV hardware:

- First series: Home 2–3 s **later** (37.2 / 38.1 / 39.1 s, against 35.1–36.4 s with them on). SM
  start times of the other services did not change.
  - The delay sat inside GUI app init, between the Disclaimer context and the end of init (the
    `vehsettingsApp.js:7050` line).
  - The log showed no DVD/TV error and no D-Bus timeout.
  - The window `UIA_DVD_InitializeGui` → `7050` took 6.3 / 9.9 / 8.7 s with the services off and
    6.2 / 5.4 / 5.6 s with them on. It had no single stall: no gap longer than 0.35 s.
- Second series (b51–b53, with `mfid` on `stage_3`): Home 33.2–34.7 s, against 33.3–34.6 s with
  them on. No difference.
- Static check: nothing waits for these services.
  - Their only clients are `libjcidvd_client.so` / `libjcitv_client.so`, loaded by the MMUI
    adapters `/jci/mmui/uia/libjciuiadvd.so`, `libjciuiatv.so` and by
    `libjciaudioaggregator.so`.
  - All client calls are async (`JCIDBUS_method_send_async_with_notify`).
  - The adapters subscribe to the SM `SERVICE_STARTED` event and mark the service available
    when it arrives.
  - The audio aggregator registers no SM callback for DVD or TV. It does for AAP, CarPlay, CD,
    radio and USB.
  - In the GUI, `dvd` and `tv` appear only as hidden entries in the `systemApp.js` source list.
- With the services on, `jcidvd`/`jcitv` start at 37.8–39.3 s, after `jciBCA` and after the
  `7050` line (34.8–35.8 s). They are not running yet in the window that was slower.
- The MMUI DVD/TV adapter lines are the same with and without the services: `UIA_TV_InitializeGui`,
  then `hand_shake_reply = NULL` at once.

The first series' ~2 s is most likely noise (3 boots per side).

### Rootfs files from eMMC

252 often-read files (114 MB) were bind-mounted from `data_persist` (relfs on eMMC) over their
rootfs paths before SM start. One boot: time from `stage_1` start to Android Auto ready was 29.9 s,
against 23.45 s with the stock paths. eMMC reads faster than FlashFX ([[FlashFX Read Path]]), but
per-file shell work before SM is expensive: a BusyBox shell loop needs ~9 ms per fork+exec on this
unit, so 252 files × 3 applet calls ≈ 8.6 s.

### Not measured

Candidates seen in the configuration, not tested one by one:

- the `blm_ira_aha` gate on `stage_wifi`, `stage_gap2` and `stage_vr`;
- the `sleep 2` calls in `jci-wifiap.sh`;
- services for absent hardware on the `jciMMUI` dependency list (`jciCD`, tuner services). The
  MMUI modules for radio, CD and AUX do not need their services to be up when they open
  ([[Boot Timeline]], "Inside jciMMUI start-up").

## Verified on

Firmware `cmu150_EU_74.00.324A`, legacy board, `sm.conf`.
