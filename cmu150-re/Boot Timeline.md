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
- 996 MHz is the fused limit: `/sys/fsl_otp/HW_OCOTP_CFG3` = `0x620302`, SPEED_GRADING bits
  [17:16] = `0b10` (996 MHz grade; `0b11` would be 1.2 GHz). `/proc/cpuinfo` revision `63000`
  (i.MX6Q/D die). The Cortex-A9 SCU configuration register (`devmem 0x00A00004` =
  `0x00005531`, bits [1:0] = `01`) reports **two CPUs in hardware**: the part is an i.MX6 Dual,
  not a Quad with cores left off (the kernel is also built for two CPUs, `kernel_max` 1). The
  `Hardware: JCI i.MX 6Quad Core CMU Board` string is only the machine name. Thermal zone `anatoptz` (cpufreq as cooling device),
  trip points 80 / 90 / 105 °C; ~31 °C read on an idle bench unit.
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

## Per-process CPU and I/O during boot (one boot, `/proc` sampled)

Method: a busybox-sh loop started from the top of `start_normal_mode.sh` (6.6 s), pinned to
cpu1, every ~2.4 s appending `/proc/stat` cpu lines, `/proc/diskstats` (`ffx01`, `mmcblk0`) and
each `/proc/<pid>/{stat,statm,io}`. One sample costs ~0.4 s CPU (≈190 processes); the sampler
itself used 5.9 s of cpu1 up to 59 s and delayed `stage_3` / `RUNNING` by ~1 s. Processes
born and gone between two samples are not seen.

- **cpu0 is saturated from 8 s to 68 s** (88–100 % busy, iowait ≤ 13 %); cpu1 runs 20–60 %
  (of which ~20 % is the sampler), except 28–40 s when both cores are at 95–100 %. After
  ~70 s cpu0 settles at 25–40 %, cpu1 at ~20 %.
- Rootfs reads (`ffx01`, NOR) run at 2–6 MB/s from 8 s to ~58 s; `mmcblk0` reads burst
  at 56–66 s (up to ~5.8 MB/s).
- **`ffxread01`** (kernel thread serving `ffx01` reads) is the largest consumer: 5.3 cpu-s up to
  `stage_3` (4.9 s of it on cpu0), 10.5 cpu-s up to 59 s. Its affinity is `0-1`
  (`Cpus_allowed_list`, not `PF_THREAD_BOUND`), but it is scheduled almost only on cpu0.
- cpu-s used from 6.6 s to 25.4 s (`stage_3`), excluding the sampler: `ffxread01` 5.34,
  `opera` 1.87 (cpu1), `jciBLMSetting` 1.45 (cpu1), `jciMMUI` 1.16, `syslog-ng` 0.94,
  `jciVBS` 0.94, SM main thread 0.63 (cpu1), `vim_app` 0.61, `settings` 0.57,
  `splashplayer` 0.52 (cpu1), `dbus-daemon` 0.49 + 0.35, `wayland-viv-compositor` 0.46 (cpu1),
  `jciUSBMGR` 0.42.
- cpu-s from 6.6 s to 58.8 s (`RUNNING` ≈ 59.7 s on this boot): `ffxread01` 10.45, `opera` 6.53,
  `jciBLMSetting` 3.50, `syslog-ng` 2.55, `jciVBS` 2.49, `jciMMUI` 2.43, `vim_app` 2.18,
  SM 1.64, `dbus-daemon` 1.52 + 1.00, `splashplayer` 1.45, `settings` 1.44, `wayland-viv` 1.21,
  `natp` 1.19, `jciCARPLAY` 1.18, `LVDS_MAIN` 0.92, two `lua` 0.91 + 0.84, `process-log` 0.79,
  `bds` 0.71. Of 71.5 cpu-s seen, 47.5 ran mostly on cpu0, 23.9 mostly on cpu1.
- Largest RSS at 152 s: `opera` 54 MB, `natp` 45 MB, `jciMMUI` 19 MB, `jciVBS` 12 MB,
  `jciSXMS` 10 MB, `bds` 10 MB.

- Same sampler with `taskset -p 2 <ffxread01>` at 5.9 s (one boot): `ffxread01` runs on
  cpu1 only (4.68 cpu-s up to `stage_3`), cpu0 busy drops to 31–78 % between 8 s and 24 s
  (cpu1 rises to 56–86 %); from 26 s to 68 s cpu0 is still saturated. Milestones vs the
  unpinned boot: `stage_3` 24.63 (25.24), `bds` 33.26 (33.92), `devicemanager` 40.42 (42.51),
  `jciAAPA` 44.34 (45.43), `jciCARPLAY` 46.86 (47.31), `RUNNING` 59.58 (59.72). The gains are
  inside the boot-to-boot spread (<1 s before `stage_3`, ~2 s after). Three pinned boots
  (sampler running): `stage_3` 24.63 / 24.51 / 25.60, `bds` 33.26 / 33.36 / 33.80, `RUNNING`
  59.58 / 59.40 / 59.08 — no measurable gain over the unpinned boots without the sampler
  (`stage_3` 24.09–25.08, `RUNNING` 57.90–59.82). Relieving cpu0 before `stage_3` does not
  shorten the boot; after `stage_3` cpu0 is saturated by SM services.

- Same sampler moving, on first sight, every thread of `natp`, `L_jciVBS`, `L_jciCARPLAY`,
  `L_jciMMUI`, `L_settings`, `L_jciblmVdt`, `L_jciVwm`, `vim_app`, `syslog-ng` and
  `dbus-daemon` to cpu1 (`taskset -p 2`, 2.4 s scan; three boots):

  | boot | `stage_3` | `bds` | `devicemanager` | `jciAAPA` | `jciCARPLAY` | `RUNNING` |
  |---|---|---|---|---|---|---|
  | 1 | 24.65 | 31.52 | 40.59 | 43.31 | 46.07 | 56.96 |
  | 2 | 24.50 | 31.36 | 41.80 | 44.19 | 46.46 | 57.58 |
  | 3 | 23.74 | 30.30 | 39.77 | 42.49 | 45.43 | 56.47 |

  Against the six boots without the sampler, `bds` starts 1–3 s earlier (30.3–31.5 vs
  32.3–34.2) and `RUNNING` ~1.5 s earlier (56.5–57.6 vs 57.9–59.8), with the sampler's own
  ~1 s cost included. `stage_3`, `devicemanager` and `jciAAPA` do not move. cpu1 becomes the
  saturated core at 27–48 s (92–100 %) and, with `natp` on it, at 57–66 s while cpu0 drops
  to 8–25 %.
- `dbus-daemon` (both instances) and the main thread of `vim_app` are back on cpu0 after
  being moved at ~5.8 s (their other threads stay on cpu1); no stock script pins them with
  `taskset`, so the affinity is reset elsewhere (not identified).

- The same move without any profiling (a scan of `/proc/*/stat` every 2 s; three boots):

  | boot | `stage_3` | `bds` | `devicemanager` | `jciAAPA` | `jciCARPLAY` | `RUNNING` |
  |---|---|---|---|---|---|---|
  | 1 | 23.60 | 30.22 | 40.04 | 42.72 | 44.58 | 55.56 |
  | 2 | 24.51 | 31.85 | 42.25 | 45.26 | 46.28 | 57.09 |
  | 3 | 23.75 | 30.39 | 40.74 | 43.65 | 44.93 | 55.70 |

  Against the six unmoved boots (medians `bds` 33.16, `jciCARPLAY` 46.27, `RUNNING` 58.73):
  `bds` ~2.8 s earlier, `jciCARPLAY` ~1.3 s, `RUNNING` ~3 s; `stage_3`, `devicemanager` and
  `jciAAPA` within the spread. The AA chain from `bds` to `jciAAPA` is not CPU-bound: it
  absorbs the earlier `bds` start.

## Critical chain to `RUNNING` (one boot, `sm.conf` + SM events)
For each service, the gate is the dependency that finished starting last. Followed back from the
last service to start (`jciMSG`), the chain is (seconds, start → started):

| Service | Start → started | Duration |
|---|---|---|
| `stage_1` | 8.97 → 9.01 | 0.04 |
| `settings` | 9.18 → 10.28 | 1.10 |
| `jciVBS` | 10.31 → 15.49 | 5.18 |
| `xsesvc2dbus_service` | 15.49 → 15.61 | 0.11 |
| `audio_control` | 15.75 → 18.03 | 2.27 |
| `audio_manager` | 18.03 → 18.45 | 0.42 |
| `jciRadio` | 18.45 → 19.76 | 1.31 |
| `jciMMUI` | 19.86 → 24.08 | 4.21 |
| `stage_3` | 24.08 → 24.09 | 0.01 |
| `bds` | 24.12 → 32.59 | 8.47 |
| `jciPA` | 32.61 → 33.87 | 1.26 |
| `jciBCA` | 33.90 → 35.52 | 1.62 |
| `mfid` | 35.55 → 35.64 | 0.09 |
| `devicemanager` | 35.69 → 39.98 | 4.29 |
| `jciCARPLAY` | 40.01 → 46.30 | 6.29 |
| `stage_vr` | 46.31 → 51.48 | 5.17 (`sleep 5s`) |
| `natp` | 51.50 → 51.51 | 0.00 |
| `jciTTSPlayer` | 51.53 → 55.05 | 3.53 |
| `jcibteca` | 55.06 → 57.46 | 2.40 |
| `jciPB` | 57.48 → 58.91 | 1.44 |
| `jciMSG` | 58.93 → 59.75 | 0.82 |

- `stage_vr` is the only pure wait on the chain; its only dependent is `natp`. Its 5 s are added
  in full to everything after it.
- `sm.conf` default is `affinity_mask="0x01"` (cpu0); 18 services set `0x02`.
- Services with no dependents (leaves, removable from the graph without blocking anything):
  `jcidvd`, `jcitv`, `jciDAB`, `jciXMMGR`, `blm_ira_pandora`, `blm_ira_stitcher`. `jciSXMS` has
  only XM services as dependents. `jciCD` is not a leaf: `jciMMUI` depends on it.

## Boot log volume (one boot, `running_log`)
- 11 585 lines (1.67 MB) up to ~60 s: Info 10 757 (93 %), Error 368, Warn 367, Note 43,
  Debug 29, Critical 12.
- Top zones: `VWM` 1335, `MMUI_SYSTEM` 680, `CPP_TIME` 580, `BLM_SETTINGS` 560,
  `MMUI_COMMON` 497, `BCA` 414, `VBS_BUS_VWM` 373, `BLM_CARPLAY` 340.
- The application-side level is set in `/config/common/log/settings.txt` (rootfs):
  `COMMON_LOG_FORMAT=SYSLOGNGEXT`, `COMMON_LOG_STRIP=True`, `COMMON_LOG_LEVEL=Info`.
  `jci-log.sh level <level>` writes that file; `jci-log.sh console disable` and
  `jci-log.sh filter` also exist.
- The live syslog-ng config is `/data/log/syslog-ng.conf` (writable `/data`). Destinations:
  `d_ram_log` (process-log, 512), `d_error_log`, and `d_console` = `/dev/console` (ttymxc1 UART,
  115200 baud). Every line is also written to the UART; ~7.7 MB transmitted by uptime 1199 s.
  The UART has no flow control, so excess lines are dropped rather than blocking.

## Cutting the `stage_vr` wait (three boots, with the cpu1 move)
The `sleep 5s` child of `stage_vr.sh` was killed as soon as it appeared (`pgrep -f stage_vr.sh`,
`pkill -P`); the script then creates `/var/run/stage_vr_ready` itself. No configuration changed.

| boot | `jciCARPLAY` | `stage_vr` | `natp` | `jciTTSPlayer` start → started | `RUNNING` | `jciCARPLAY` → `RUNNING` |
|---|---|---|---|---|---|---|
| wait cut 1 | 46.38 | 47.19 | 47.23 | — | 54.72 | 8.3 |
| wait cut 2 | 46.74 | 47.51 | 47.56 | 47.59 → 52.18 (4.6) | 54.35 | 7.6 |
| wait cut 3 | 43.25 | 44.72 | 44.76 | 44.78 → 50.74 (6.0) | 53.45 | 10.2 |
| move only | 43.33 | 48.55 | 48.58 | 48.60 → 52.09 (3.5) | 54.22 | 10.9 |

- With the move only, `jciCARPLAY` → `RUNNING` is 10.8–10.9 s on all four boots; with the wait cut
  it is 7.6–10.2 s.
- `natp` reports started within 0.05 s, but `jciTTSPlayer` (depends on `natp`, `ping_timeout=30000`)
  then takes 4.6–6.0 s instead of ~3.5 s: part of the removed wait reappears as `natp`
  initialisation. Net effect on `RUNNING`: about 1.5 s (mean 54.2 s against 55.6 s), within the
  boot-to-boot spread of the earlier chain (`devicemanager` 4–8 s).
- No board reset, UI up on all three boots. `natp`, `jciTTSPlayer` and `jciVUI` carry
  `reset_board="yes"`.

## Reading
- Kernel to SM start ≈ 9 s; SM start to HMI stage (`stage_3`) ≈ 15.6 s; SM start to SM
  `RUNNING` ≈ 49.5 s. The SM phase dominates; most of it is service start-up serialised behind
  `jciMMUI` and the stage barriers.
- Candidate levers visible in the config (not measured individually): the `stage_vr` fixed
  sleep, the `blm_ira_aha` gate on Wi-Fi/VR, the `sleep 2` calls in `jci-wifiap.sh`, and
  services for absent hardware on the `jciMMUI` dependency list (`jciCD`, tuner services).
