---
tags: [cmu150, boot, sm, timing]
---

# Boot Timeline

When the System Manager (SM) stages and the big services start in a normal boot, and what the
stage graph waits on. Firmware `cmu150_EU_74.00.324A`, legacy board (`/jci/sm/sm.conf`).
Part of [[CMU150]]; NOR/boot-loader side in [[Boot Chain and Fail-Safe]], SM tool in
[[SM Service Control (smctl)]].

Sources: one boot's SM event dump (`PID,Component,Event,Timestamp`, timestamps = seconds since
kernel start), the kernel log of the same boot, and stock `sm.conf`. One capture only; expect a
few seconds of spread between boots. Time before the kernel (boot ROM, bootstrap, IBC) is not
visible in these clocks.

## Milestones (one boot, seconds since kernel start)
| t (s) | Event |
|---|---|
| 1.08 | SPI-NOR partitions registered |
| 2.50 | relfs VOL0 (`/dev/ffx01p1`, root) mounted |
| 3.34 | VOL1 (`/dev/ffx01p4`, `/data`) mounted |
| 3.48 | VOL2 (`/dev/mmcblk0p2`, `/data_persist`) mounted |
| 9.03 | SM server init/starting |
| 9.23 | `stage_1` |
| 10.6–15.8 | `jciVBS` (5.2 s), `jciSXMS` (3.7 s) |
| 12.61 | `stage_2` |
| 14.63 | `Opera` started |
| 16.0–18.8 | audio chain: `audio_amplifier`, `audio_control`, `audio_manager`, then tuner/CD/DAB/XM services |
| 20.40 → 24.65 | `jciMMUI` start → started (4.3 s) |
| 24.65 | `stage_3` (HMI displayed) → `stage_navi`, `emmc_mount`, `bds` |
| 25.4 → 25.58 | VOL3 (`/dev/mmcblk0p1`, `/mnt/resources`) mounted by SM `emmc_mount` |
| 25.66 | splash player shows first frame |
| 25.71 | `stage_bt`, `stage_usb` |
| 24.70 → 33.03 | `bds` (Bluetooth, 8.3 s, longest single service) |
| 25.94 → 32.51 | `start_network` (6.6 s) |
| 35.77 | `stage_ir` |
| 39.04–39.10 | `stage_gap2`, `stage_wifi` |
| 38.83 → 46.04 | `jciCARPLAY` (7.2 s) |
| 46.04 → 51.21 | `stage_vr` (`sleep 5`) |
| 58.53 | SM `RUNNING` (all autorun services up) |

Event counts in that boot: 107 `SM_SVC_CMD_START`, 112 `STARTED`, 64 handshakes.

## Longest start→started durations
`bds` 8.3 s · `jciCARPLAY` 7.2 s · `start_network` 6.6 s · `stage_vr` 5.2 s (fixed sleep) ·
`jciVBS` 5.2 s · `jciVwm` 5.0 s · `jcinavi` 4.8 s · `jciblmVdt` 4.4 s · `jciMMUI` 4.3 s ·
`jciAAPA` 3.9 s · `jciUSBM` 3.9 s.

## Stage graph (`sm.conf`)
- 117 `<service>` entries; 91 carry `reset_board="yes"` (incl. `Opera`, `jciMMUI`, `bds`,
  `jciUSBMGR`, `jciCARPLAY`, `jciAAPA`, the audio chain), 26 `reset_board="no"` (incl. all `stage_*`,
  `emmc_mount`, `usb_drivers`, `start_network`).
- Stage dependencies:

| Stage | Depends on |
|---|---|
| `stage_2` | `jciBLMSettings` |
| `stage_3` | `jciMMUI` |
| `stage_navi` | `stage_3` |
| `stage_usb` | `stage_3`, `jciUSBMGR`, `emmc_mount` |
| `stage_bt` | `stage_3`, `emmc_mount` |
| `stage_ir` | `stage_3`, `jciBCA` |
| `stage_wifi` | `stage_3`, `blm_ira_aha` |
| `stage_gap2` | `stage_3`, `blm_ira_aha` |
| `stage_vr` (`args="5s"`) | `stage_3`, `blm_ira_aha`, `jciCARPLAY` |
| `emmc_mount` | `stage_3` |

- `jciMMUI` depends on `jciBUCPSA`, `jciBLMSettings`, `jciRadio`, `jciauxin`, `jciCD`, so the
  tuner/CD services sit on the HMI critical path. `Opera` depends on `jciAppsdk`, `jciDbapi`.
- `stage_vr.sh` is `sleep "$1"` then `touch /var/run/stage_vr_ready`; with `args="5s"` it adds a
  fixed 5 s before the VR-gated services.
- `stage_wifi`, `stage_gap2` and `stage_vr` all wait for `blm_ira_aha` (Aha streaming client).
- `/jci/scripts/jci-wifiap.sh` contains four `sleep 2` calls (on different code paths).
- The resources partition (`mmcblk0p1`) is mounted only after `stage_3`, by SM service
  `emmc_mount` (`/usr/sbin/emmc_mount.sh`); `/data_persist` (`mmcblk0p2`) is already mounted
  before SM starts.

## Which SM config runs, and where the events come from
`/usr/bin/autostart` (normal-mode branch) sources `/jci/scripts/start_normal_mode.sh`, starts the
logger, `sleep 1`, then runs `/jci/scripts/get_board_type.sh`. That script reads GPIOs
`/sys/class/gpio/DI_TP1056/value` and `DI_TP1029/value`; if either is `1` it exits 2 ("WCP, new
hardware") and SM starts with `sm_WCP.conf`, otherwise with `sm.conf`:
`taskset 0x02 /jci/sm/sm -f /jci/sm/sm.conf -e /tmp/smevents.txt`. On a legacy board both GPIOs
read `0` and the running SM shows `-f /jci/sm/sm.conf`.

The event dump above is that `-e` file: `/tmp/smevents.txt`, CSV `PID,Component,Event,Timestamp`
(seconds since kernel start), written fresh every boot on tmpfs, so it is lost at the next
reboot. Copy it off after each boot to keep a timing record; no extra tooling is needed.

`sm_WCP.conf` differs from `sm.conf` by `wcp` / `new_hw` / `-wcp` args on `jciMMUI`, `jciUSBMGR`,
`jciCARPLAY`, `jciAAPA`, `bds`, `jciBCA`, `jcinavi`, `GPS=UBLOX` on `jciLDS` (plus a `stage_navi`
dependency), an extra `emmc_mount` dependency, and `args=""` instead of `args="UCP"` on
`start_network`. Edits to SM services have to be made in both files to cover both boards.

## CPU
- Two Cortex-A9 cores, both online. cpufreq governor `userspace`, both cores at `996000` kHz;
  available `996000 792000 396000`.
- SM itself is pinned to core 1 (`taskset 0x02`), but the `<services>` default in `sm.conf` is
  `affinity_mask="0x01"`, so services run on core 0 unless they set `0x02`. Stock `0x02`
  services: `jciBLMSettings`, `devices`, `jciNativegui`, `jciBUCPSA`, `jciVideoctrl`, the splash
  players, `Opera`, `dsp_config`, `audio_config`, `audio_start`, `aap_service`, `carplayd`, `NNG`,
  `JVMM`, `jciTds`, `start_wifi`.
- Running unit, ~3 h after boot: 106 processes allowed only on core 0, 15 only on core 1, 1 on
  both; resident memory about 500 MB on core-0 processes vs 124 MB on core-1 processes (Opera
  56 MB is the largest, on core 1). `/proc/stat` user+system time: core 0 about 351 k jiffies,
  core 1 about 57 k, i.e. core 0 does roughly six times the work.

## Second capture (legacy board, unit with third-party packages installed)
Close to the first: `stage_1` 9.91, `stage_2` 13.11, `Opera` 15.28, `jciMMUI` 20.86 → 24.92,
`stage_3` 24.93, `emmc_mount` 24.97 → 26.16, `start_network` 26.34 → 33.82, `bds` 24.97 → 34.15,
`stage_ir` 37.06, `jciCARPLAY` 42.42 → 47.16, `stage_vr` 47.17 → 52.37, SM `RUNNING` 59.77.

The Wi-Fi/CarPlay chain in that boot, each step waiting on the one before:
`bds` started 34.15 · `jciBCA` 35.84 → 37.00 · `stage_ir` 37.06 · `blm_ira_aha` 37.25 → 40.34
(3.1 s) · `stage_wifi` / `stage_gap2` 40.37 · `jciCARPLAY` 42.42 → 47.16 · `stage_vr` 52.37.
`blm_ira_aha` itself depends on `stage_ir`, `offboard_service`, `bds`, `audio_manager`. In the same
window, `irsstitcher_server` (37.19 → 37.98), `blm_ira_stitcher` (38.00 → 39.92) and
`blm_ira_pandora` (38.03 → 40.21) start on core 0 alongside `blm_ira_aha`.

## Five-boot baseline (legacy board, `sm.conf`)
Five consecutive SafeReboot (warm) boots, each left up for more than 150 s, `/tmp/smevents.txt`
copied after each. Same unit as the second capture (third-party packages installed). Seconds since
kernel start; spread = max − min.

| Milestone | Median | Min – max |
|---|---|---|
| SM `STARTING` | 9.07 | 8.95 – 9.73 |
| `stage_2` | 12.59 | 12.48 – 13.29 |
| `Opera` started | 14.67 | 14.40 – 15.35 |
| `jciMMUI` start → started | 20.07 → 24.24 | 24.09 – 25.06 |
| `stage_3` (HMI displayed) | 24.26 | 24.10 – 25.08 |
| `start_network` started | 32.87 | 29.89 – 33.82 |
| `bds` started | 33.17 | 32.30 – 34.21 |
| `stage_ir` | 35.96 | 35.11 – 37.30 |
| `blm_ira_aha` start → started | 36.17 → 39.46 | 38.37 – 40.32 |
| `stage_wifi` | 39.49 | 38.40 – 40.36 |
| `jciCARPLAY` start → started | 40.06 → 46.24 | 45.27 – 47.31 |
| `stage_vr` started | 51.44 | 50.46 – 52.56 |
| SM `RUNNING` | 58.52 | 57.90 – 59.82 |

Spread is under 1 s up to `stage_3` and about 2 s after it (`start_network` up to 3.9 s). A
single lever has to move a milestone by more than that spread to show in five boots.

## What the screen shows (one boot, GUI context messages)
MMUI sends each screen change to the GUI as `{"msgType":"ctxtChg","uiaId":…,"ctxtId":…}` (logged by
`GUICORE … GUIIFM_SendDataToGUI` in `/var/log/running_log/log`). Uptime from the log's wall clock
(offset from `date` + `/proc/uptime` read together):

| t (s) | Screen |
|---|---|
| 24.89 | `system` / `IntroAnimation` (boot screen) |
| 30.37 | `system` / `Disclaimer` |
| 41.86 | `system` / `HomeScreen` (after `UIA_SYSTEM_Event_Timeout` from the GUI) |

`systemApp.js` (`_disclaimerTime`) starts a 3500 ms timer when the Disclaimer shows and sends
`Timeout` to MMUI when it fires; the timer pauses while another screen covers the Disclaimer. In
this boot the `Timeout` reached MMUI 11.5 s after the Disclaimer context was sent, not 3.5 s.
What held it is not established. `Global.Yes` on the Disclaimer sends the same exit early.
`running_log` rotates about every 1.5 min (ten `log_NNNN.tar.gz` kept), so boot lines survive only
for roughly the first 15 minutes of uptime.

## Fixed waits in boot-path scripts
From the stock rootfs (`usr/bin/autostart`, scripts named in `sm.conf`):
- `usr/bin/autostart:78` — `sleep 1` before `get_board_type.sh`, i.e. before SM starts (normal mode).
- `jci/scripts/stage_vr.sh:2` — `sleep "$1"` (`args="5s"`).
- `jci/scripts/jci-wifiap.sh` — `sleep 2` at lines 62, 76, 92, 105; `sleep 1` at 185.
- `jci/scripts/get_board_type.sh` — `sleep 1` only on a failed GPIO read (retry loop).
- `usr/sbin/emmc_mount.sh:10` — `waitfor /sys/block/${MMC_DEV}/dev 30000` (bounded poll, not a fixed wait).
- `jci/scripts/wait_core_finished.sh` — 1 s poll loop on `/var/lock/*.core`, up to 300 s.
- Timers inside binaries are not visible here, e.g. BCA logs `Incoming Connection Wait timer started
  with timeout 5000 milli sec` twice around 35.9–36.0 s.

## Reading
- Kernel to SM start ≈ 9 s; SM start to HMI stage (`stage_3`) ≈ 15.6 s; SM start to SM
  `RUNNING` ≈ 49.5 s. The SM phase dominates; most of it is service start-up serialised behind
  `jciMMUI` and the stage barriers.
- Candidate levers visible in the config (not measured individually): the `stage_vr` fixed
  sleep, the `blm_ira_aha` gate on Wi-Fi/VR, the `sleep 2` calls in `jci-wifiap.sh`, and
  services for absent hardware on the `jciMMUI` dependency list (`jciCD`, tuner services).
