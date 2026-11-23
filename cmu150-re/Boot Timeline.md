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

The Disclaimer's displayed callback (which sends `DisposeIntroVideo` and starts the timer) runs only
once the GUI has finished initialising its apps: in five boots `DisposeIntroVideo` came 0.0–0.5 s
after the `vehsettingsApp.js` "context/template not initialized" error line, 5.8–8.9 s after the
Disclaimer context. `DisposeIntroVideo` is when NATGUICTRL switches from the native `Newlooplogo`
surface to `JCI_OPERA_PRIMARY`, so the boot animation lasts until the GUI is ready. `Timeout` sent
in the same callback (no 3500 ms timer) gives `HomeScreen` 0.2–0.9 s after `DisposeIntroVideo`.
`running_log` rotates about every 1.5 min (ten `log_NNNN.tar.gz` kept), so boot lines survive only
for roughly the first 15 minutes of uptime.

## What gates the Disclaimer context (3 boots: stock b10, two with GUI tweaks)
- MMUI starts the SM service `tranlogo` on demand right before it sends `IntroAnimation`
  (`UIA_SYSTEM_GetStartUpSettings` "tranlogo is started"). `tranlogo` is
  `/usr/bin/splashplayer -q -p /var/run/tranlogo_ready_flag -t TRANLOGO_SURFACE -z 700 /jci/resources/TranLogo.ivf`
  (`sm.conf`, `autorun="no"`, cpu1). `TranLogo.ivf` is 1 240 681 bytes.
- `tranlogo` runs 4.4–4.8 s (start 25.46–26.76 → `Svc tranlogo stopped!` 30.07–31.54).
  MMUI sends `ctxtChg Disclaimer` 0.6–1.9 s after that stop, in every boot.
- So the boot animation, not the GUI, sets the earliest Disclaimer time. After the Disclaimer,
  MMUI stops `quicktranlogo` (`LoopLogo.ivf`, looped), which is already stopped, and NATGUICTRL
  keeps showing the `Newlooplogo` surface until the GUI takes over.
- The IVF files are VP8, 800×480, with a 90 000 timebase. `TranLogo.ivf` has 120 frames,
  `TranLogoEnd.ivf` 30, `LoopLogo.ivf` 54. `newLoopLogo.ivf` has 60 frames and a 30 fps header.
  `splashplayer` plays at `-r <fps>` (default 30), so `TranLogo` lasts 120/30 = 4.0 s, plus
  start-up. Other options: `-l` loop, `-q` quit at end, `-p` pid file, `-t` surface title,
  `-x -y -w -h -z` window position, size and z-order, `-f` format (9 = VP8).
- `TranLogo` is the Mazda logo turning into a pre-rendered picture of the Home menu: the last frames
  are video, not the live GUI.

## `natp`
`/jci/natp/bin/natp -f /jci/natp/config/natp.cfg -s natp_config -k VerboseLevel=0` is the Nuance
speech platform. Its `natp.cfg` channels are HostAgent, MonitorService, GDM, AudioMatrixService,
AudioRecordingService, PrompterService, SAIP/SAOP/SAS, FileService and VoconService (Vocon ASR,
TTS decoder). In `sm.conf` it depends on `emmc_mount` and `stage_vr`, and carries `reset_board="yes"`.
`jciTTSPlayer` and the voice UI depend on it. It starts after HomeScreen (≈45–51 s).

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
- Leaf in the SM graph does not mean free to drop. With `jcidvd` and `jcitv` set `autorun="no"`
  (both SM files, on a unit with no DVD/TV hardware), the Home screen came 2–3 s **later**
  (3 boots: 37.2 / 38.1 / 39.1 s, against 35.1–36.4 s with them on). SM start times of the other
  services did not change. The delay sat inside GUI app init, between the Disclaimer context
  and the end of init (`vehsettingsApp.js:7050` line). No DVD/TV error or dbus timeout appeared in
  the log, so the waiting party is not identified.
- Who uses `jcidvd` / `jcitv` (rootfs search for `com.jci.dvd`, `com.jci.tv` and their client
  libraries): only `libjcidvd_client.so` / `libjcitv_client.so`, loaded by the MMUI adapters
  `/jci/mmui/uia/libjciuiadvd.so`, `libjciuiatv.so` and by `libjciaudioaggregator.so`. All client
  calls are async (`JCIDBUS_method_send_async_with_notify`). The adapters subscribe to the SM
  `SERVICE_STARTED` event and mark the service available when it arrives. The audio aggregator
  registers no SM callback for DVD or TV (it does for AAP, CarPlay, CD, radio, USB). In the GUI, `dvd`
  and `tv` appear only as hidden entries in the `systemApp.js` source list.
- With the services on, `jcidvd` / `jcitv` start at 37.8–39.3 s (after `jciBCA`). That is after
  `vehsettingsApp.js:7050` (34.8–35.8 s), so they are not yet running in the window that was slower.
  The MMUI DVD/TV adapter lines are the same with and without the services
  (`UIA_TV_InitializeGui`, then `hand_shake_reply = NULL` at once). The slower window
  (`UIA_DVD_InitializeGui` → `7050`: 6.3 / 9.9 / 8.7 s off, 6.2 / 5.4 / 5.6 s on) has no single stall:
  no gap longer than 0.35 s, so the time is spread over the whole window. No component that waits
  for DVD/TV was found. With 3 boots per side, the ~2 s difference may be noise.

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

## Screens per boot vs. the cpu1 move
`ctxtChg` times (uptime, s) from `running_log`, all boots of the series above:

| boots | `IntroAnimation` | `Disclaimer` | `HomeScreen` |
|---|---|---|---|
| unmoved (5 boots: plain, sampler, `ffxread01` pinned) | 24.9–26.9 | 30.4–32.1 | 41.0–41.9 |
| cpu0-heavy services moved to cpu1 (8 boots) | 23.9–25.6 | 29.6–31.1 | 46.0–53.4 |

- The move that brings `RUNNING` ~3 s earlier delays `HomeScreen` by 4–12 s. `opera` already runs
  on cpu1 (6.8 CPU-s there in the first 62 s); `jciMMUI`, `jciVwm`, `syslog-ng`, `natp` and others
  moved next to it compete with it while the Disclaimer timer (3500 ms in `systemApp.js`) runs.
- Unmoved, the Disclaimer still lasts 9.3–10.0 s against its 3.5 s timer.

## Reading
- Kernel to SM start ≈ 9 s; SM start to HMI stage (`stage_3`) ≈ 15.6 s; SM start to SM
  `RUNNING` ≈ 49.5 s. The SM phase dominates; most of it is service start-up serialised behind
  `jciMMUI` and the stage barriers.
- Candidate levers visible in the config (not measured individually): the `stage_vr` fixed
  sleep, the `blm_ira_aha` gate on Wi-Fi/VR, the `sleep 2` calls in `jci-wifiap.sh`, and
  services for absent hardware on the `jciMMUI` dependency list (`jciCD`, tuner services).

## CPU by process during boot (fw 74.00.324A, measured)

Cumulative utime+stime from `/proc/<pid>/stat`, read at the first SSH login (uptime 57–63 s), two
boots. HomeScreen was at 32–33 s with GUI and SM tweaks. Across 2 cores, about 78% of the CPU was
busy up to that point (`/proc/uptime`: idle 26–30 s of 59–64 s × 2).

| process | CPU s | note |
|---|---|---|
| `ffxread01` | 12.1 | FlashFX (Datalight NAND) read kernel thread, all system time |
| `opera` | 9.1–9.8 | HMI GUI |
| `natp` | 2.9–5.7 | |
| `L_jciBLMSetting` | 3.1–3.5 | |
| `syslog-ng` + `process-log` | 3.5 | system log path |
| `L_jciVBS`, `L_jciMMUI`, `vim_app`, `init` | 2.3–2.8 each | |

- The system log carries about 8 300 lines (1.3 MB) in the first ~55 s; 91% of them are Info.
  Busiest sources: BLM_SETTINGS, VWM, MMUI_SYSTEM, BLM_CARPLAY, CPP_TIME. After boot the rate
  drops to about 5 KB/s, and syslog-ng + process-log then use about 1.7% of one core.
- `running_log` rotates at 512 KB into `log_NNNN.tar.gz`.
- The GUI's own `Log.js` output is not a boot cost: only 10–43 lines per boot pass its level gate.

## Which boot logo plays: `/data/unexpectedshutdownflag`

`/sbin/init_cmu` reads `/data/unexpectedshutdownflag` (one byte) before the Service Manager starts, and picks the loop logo from it:

| flag | init log | loop logo | what is seen |
|---|---|---|---|
| `1` | `<I> Detected CMU reboot in previous cycle` | `splashplayer -l -t NEWLOOPLOGO_SURFACE -z 1700 -p /var/run/newlooplogo.pid /jci/resources/newLoopLogo.ivf` | logo + spinner until the Home screen; the `tranlogo` animation (z 700) and the Disclaimer still run but are covered |
| other | — | `splashplayer -l -t LOOPLOGO_SURFACE -z 500 -p /var/run/looplogo.pid /jci/resources/LoopLogo.ivf` | loop logo, then the `tranlogo` animation and the Disclaimer on top |

- Reader: `cb_looplogo` in `/sbin/init_cmu` (also `init_cmu_rr`) reads one byte; any byte other than `'0'` (or a missing file: no) selects the newlooplogo path. No other binary reads the file.
- Writer: `SYSTEM_SMM_UnexpectedShutdown(int)` in `/jci/system/svcjciblmsystem.so` writes `%d` + `sync()`, only when the value differs from the file.
- SMM state enum (`SystemStateMachine` name table): 0 `InvalidState`, 1 `SYSTEM_InitState`, 2 `SYSTEM_NormalState`, 3 `SYSTEM_IdleState`, 4 `SYSTEM_DelayState`, 5 `SYSTEM_StandbyState`, 6 `SYSTEM_PoweringDownState`, 7 `SYSTEM_TestState`, 8 `SYSTEM_VipReflashState`, 9 `SYSTEM_MAP_SleepState`, 10 `SYSTEM_MAP_ColdStartState`.
- Writes, by caller:

| caller | writes | when |
|---|---|---|
| `system_smm_PrepareForState`, target `NormalState` (2) | `1` | every boot, when the system reaches Normal |
| `system_smm_PrepareForState`, target `StandbyState` (5) | `0` | orderly ACC off (display ACC-off or sleep, audio off) |
| `DbusServerClientModule::genericReboot` (dbus `GenericReboot`) | `0` | after `SYSTEM_HSRM_StartBoardReset`/`StartShutdown` returns 100 |
| `factoryReset`, `system_smm_VIPReflash_EventHandler`, `SYSTEM_SMM_EnterVIPReflashMode` | `0` | factory reset, VIP reflash |

- `DbusServerClientModule::safeReboot` (dbus `SafeReboot`) never writes the flag: it calls `SetEmergency`, `SetDisplayOff`, `SetAudioOff`, `SYSTEM_HSRM_StartBoardReset(0,1,0)`. So `1` from Normal stays, and the next boot shows newlooplogo. A power cut (bench PSU off) also leaves `1`. Only an orderly ACC off (Standby) gives the LoopLogo path.
- `GenericReboot` with `mode_flags` = 0 decodes to the same `StartBoardReset(0,1,0)` with display and audio off, plus the `0` write. Flag bits: `1` = keep display on (also passed to `StartBoardReset`), `2` = clears arg 2, `4` = shutdown instead of reset (refused unless bit 2 is set and bit 1 is clear), `8` = arg 3.
- In the newlooplogo path `jciNativegui` re-shows the surface on idle (`NGC_CheckNewLooplogoAnimation`, `OnIdle: Showing Newlooplogo surface...`). MMUI stops it when the Home screen comes (`uiaSystem_Stop_NewLoopLogo`: `kill -TERM $(cat /var/run/newlooplogo.pid)`).
- So on this path, a shorter `tranlogo` cannot be seen and only matters by when MMUI sends the Disclaimer.
- Measured (stock, `GenericReboot int32:0`): log `read /data/unexpectedshutdownflag value is [0]`, LoopLogo path, Disclaimer context 33.20 s, `DisposeIntroVideo` 33.56 s, Home 37.06 s. The newlooplogo path on the same unit gave Home 37.2 s, so the flag changes only what is seen, not the timing.

## Critical chain to Android Auto (two boots, `sm.conf` + SM events + running log)

Seconds since kernel start. The phone was plugged in during the boot (b40, with GUI tweaks: Home 30.6 s). b43 (stock, no phone) has the same SM chain within 0.5 s.

| step | b40 | b43 |
|---|---|---|
| `bds` start → started (`ready_flag /tmp/bds_ready_flag`) | 23.59 → 31.29 | — |
| `jciPA` start → started (needs `bds`) | 31.31 → 32.30 | — |
| `jciBCA` start → started (needs `bds`, `jciPA`) | 32.32 → 33.20 | 32.63 → 33.84 |
| `mfid` (needs `jciUSBMGR`, `jciBCA`; exits after ~0.4 s) | 33.22 | 33.85 |
| `devicemanager` start → started (needs `mfid`) | 33.36 → 37.90 | 33.88 → 38.05 |
| `aap_service` (needs `devicemanager`) | 37.93 | 38.07 |
| `jciAAPA` start → started (needs `devicemanager`, `aap_service`, …) | 37.94 → 40.14 | 38.08 → 41.45 |
| USB manager `Started accessory mode - OK`, phone re-enumerates `18d1:2d00` | 38.4 / 39.0 | — |
| AAP `Starting AAP service` → `Session created` | 40.4 → 43.4 | — |
| `ctxtChg androidauto AndroidAutoUI` (after `RequestBTPairing: already paired`) | 45.7 | — |

- The AA projection waits on the Bluetooth stack: `bds` (7.7 s from start to ready flag, of which ~1.4 s is the BlueGo UART init) → `jciPA` → `jciBCA` → `mfid` → `devicemanager` (4.5 s) → `jciAAPA`. The USB hubs are up at ~14.6 s (`jciUSBMGR` started) and the phone is enumerated long before the switch to accessory mode.
- In the running log the first `BDS` line comes ~3 s after SM starts `bds`.
- From `jciAAPA` start to the AA screen: ~7.8 s (AOA switch ~0.5 s, AAP session create ~3 s, BT pairing check ~1.8 s).
- Both cores are 90–100 % busy from ~25 s to ~50 s (`/proc/stat` sampled, one boot). The chain is slow from waiting, not from its own work: `bds` used under 0.6 s of CPU (utime + stime) in its 7.7 s start.
- `devicemanager` (`/jci/devicemanager/libdevicemanager.so`) logs nothing to the running log during its start (its own log targets are `/tmp/RACE_LOG.txt`, `/tmp/devmgr_debug.txt`). It handles both iAP (CarPlay, iPod) and AOA (Android Auto) devices.
- `mfid` (`/usr/bin/mfid`) is the Apple MFi auth-chip daemon: I2C (`/dev/i2c*`, `Device I2C address`), unix socket `/tmp/MFISocket`; links only `libprotobuf-c`, `librt`, `libpthread`, `libc`, so it uses no Bluetooth or D-Bus. Its `jciBCA` dependency in `sm.conf` / `sm_WCP.conf` is ordering only. Users of the MFi socket/library: `blmjcicarplay.so`, `ipoddev_full_auth`, `libmfi.so`, `libCarPlayPlatformMFI.so`.
- Services downstream of `mfid` (transitive `sm.conf` dependencies): `devicemanager` → `aap_service`, `jciCDRP`, `jciUSBMS` (→ `jciUSBM`, `jciITTS` → `jciITT`), `jciCARPLAY` (→ `stage_vr` → `natp` → `jciTTSPlayer`, `jciVUI` → `kapture`; → `jcibteca` → `jciPB` → `jciMSG`), `jciAAPA`.
- Besides `devicemanager`, `jciAAPA` needs `jciRM` (~25.5 s) and `jciUpdatea` (~28.8 s). So with `devicemanager` up early, `jciAAPA` still cannot start before ~29 s.
- AAP session create (~2.9 s, `Calling service create session` → `Session created`): XML config and certs ~0.3 s, then RACE init (gracenote client, DB restore), and a `gst-plugin-scanner` run (GStreamer 0.10 registry rebuild, `Failed to load plugin` warnings) during the last ~1.5 s. `gst-plugin-scanner` runs on every boot in the logs (b39, b40, b43; two scanner processes in b40).
- GStreamer registry location: the registry is saved, but to tmpfs (`/tmp/root/.gstreamer-0.10/`), so it is lost at every boot; `HOME=/` is not used. See "GStreamer plugin registry is rebuilt on every stock boot" below.
- The USB Android Auto device drops accessory mode when the AA services start late (b46, Home 32.4 s). The device was a wireless Android Auto USB adapter (serial `LY0123456789` in `UsbMgr.cpp searchAndroidSetting`; the same device in b40), not a phone on a cable. USB switched it to accessory mode at ~32.8 s, but `jciAAPA` was up only at 41.5 s. At 41.45 s (kernel) the device detached, re-enumerated, and accessory mode was set again at ~53.0 s. `AAP_PROJECTION_SETUP` followed at ~54.4 s. In b40 (accessory at 38.4 s, session at 43.4 s) there was no second cycle. So with this adapter, a gap of ~9 s between accessory mode and `jciAAPA` costs one full reconnect (~11 s). Whether the drop is the adapter's own timeout or a phone-link event is not known; a phone on a USB cable may behave differently.
- Running-log timestamps are the time the line was written, in batches: a kernel line stamped 41.45 s was written ~1 s later. Treat log times as ±1 s.

## Screen-dark offset (stopwatch vs log, b46)

The stopwatch was started when the screen went dark on a `GenericReboot(0)`.

| Event | Stopwatch (s) | Log, since kernel start (s) | Difference (s) |
|---|---|---|---|
| Home ready (`Recv ctxtChg:HomeScreen`) | 37.6 | 32.4 | 5.2 |
| Boot animation visible / `tranlogo is started` | 34.1 | 25.2 | 8.9 |
| Android Auto on screen / `AAP_PROJECTION_SETUP` | 63.8 | 54.4 | 9.4 |

Screen dark to kernel start is ~5 s (Home is the sharpest event). The animation becomes visible ~3.7 s after `tranlogo is started`. The Android Auto picture comes ~4 s after `AAP_PROJECTION_SETUP`. A Home that still responds slowly was reported at 41.8 s on the stopwatch.

## `mfid` on `stage_3`, DVD/TV off (b48–b53, fw 74.00.324A, GUI tweaks, wireless AA USB adapter)

`mfid`'s `jciBCA` dependency was changed to `stage_3` (both SM files). b51–b53 also had `jcidvd` and `jcitv` set `autorun="no"`. Seconds since kernel start:

| | `mfid` | `devicemanager` | `jciAAPA` | AAP session | Home | `AAP_PROJECTION_SETUP` |
|---|---|---|---|---|---|---|
| stock deps (b46, b47) | 34.4–34.6 | 38.5–39.4 | 41.5–41.7 | 42.3–42.5 | 32.4–33.5 | 54.4–57.7 |
| `mfid` → `stage_3` (b48–b50) | 24.8–25.4 | 30.7–34.0 | 34.1–37.0 | 35.0–37.6 | 33.3–34.6 | 57.8 / 46.4 / 42.2 |
| + DVD/TV off (b51–b53) | 24.6–25.3 | 31.7–32.8 | 34.7–35.8 | 35.6–36.1 | 33.2–34.7 | 55.2 / 55.6 / 42.1 |

- The Android Auto host side (`jciAAPA`, AAP session) is ready ~6–7 s sooner. Home did not move.
- `jcidvd` / `jcitv` off made no difference to Home (33.2–34.7 s against 33.3–34.6 s). This agrees with the static finding that nothing waits for them.
- With a wireless Android Auto USB adapter, the time to projection is set by the adapter: it detaches and stays off the bus ~10–13 s at a point that varies per boot. Projection came at ~42 s when the adapter connected once (b50, b53), and at 46–58 s when it went through an extra detach cycle. A phone on a USB cable was not tested.

## Wireless Android Auto USB adapter vs AAP session timing (b47–b53)

Adapter: USB name `smartBox`, serial `LY0123456789`, presents as an Android device, is switched to AOA (`18d1:2d00`) by `CPP_USBMGR`. Seconds since kernel start; "AOA dev" = re-enumerated in accessory mode, "created" = `Service:Session created`.

| Boot | AOA dev | AAP `session start` | created | adapter detach | projection |
|---|---|---|---|---|---|
| b50 | 36.6 | 37.2 | 41.9 | — | 42.2 |
| b53 | 36.2 | 37.7 | 41.5 | — | 42.1 |
| b49 | 23.0 → detach 31.6, again 43.3 | 43.3 | 45.9 | 31.6 | 46.4 |
| b47 | 36.7 | 43.0 | 45.3 | 46.3 | 57.7 |
| b51 | 32.3 | 36.7 | 40.9 | 41.5 | 55.2 |
| b52 | 32.6 | 37.7 | 42.4 | 41.8 | 55.6 |

- In accessory mode the adapter waits ~8.6–9.8 s for the head unit's AAP traffic, then detaches and stays off the bus ~10–13 s. (One boot, b48, kept it for ~23 s.)
- `jciAAPA` does pick up an adapter that attached before it started (`FindAndroidAutoDevice: AndroidAuto device is already connected` → `StartSession`). The loss comes from the first `session start` → `Session created` step: 2.3–4.7 s, in the CPU-heavy window. It includes the per-boot `gst-plugin-scanner` run.
- Once a session exists, a re-attached adapter reaches projection ~0.5 s after `session start`.
- Projection comes first time when `Session created` lands within ~8 s of the adapter's AOA re-enumeration.

## GStreamer plugin registry is rebuilt on every stock boot (fw 74.00.324A)
- GStreamer 0.10 on glib 2.22 takes its home directory from the passwd entry, not from `$HOME` (`HOME=/` for SM services is ignored). That home is `/root`, a stock symlink to `/tmp/root` (tmpfs, created by `init_cmu`).
- So the registry is written each boot to `/tmp/root/.gstreamer-0.10/registry.arm.bin` (~66 KB) and lost at the next boot; `gst-plugin-scanner` rescans every plugin at ~40 s, during the Android Auto session setup.
- `GST_REGISTRY` (honoured by libgstreamer-0.10) sets the registry file path. SM services inherit their environment from `/etc/profile` (a regular rootfs file; `/etc/passwd` is the link to `/config-mfg`). With `export GST_REGISTRY=<file on /data_persist>` there, the scanner ran only on the first boot.
- Measured with a wireless AA USB adapter (3 boots, cached registry): AA projection at 37.7 / 39.4 / 38.5 s uptime (avg 38.5), against 42–55 s with the scanner.

## Per-boot caches and tmpfs content (fw 74.00.324A, one boot, ~5 min uptime)
- `/tmp` (tmpfs, 93 MB) holds ~2.8 MB after boot. `/var`, `/root`, `/mnt`, `/media` and `/etc/resolv.conf` are rootfs symlinks into `/tmp`.
- Files rebuilt every boot, with creation time since kernel start: `/tmp/devmgr_config_edit.xml` 15 s (VBS `sed` from a template), `/tmp/radio_database` 16 s (52 KB copy of `/data/radio/radio_database`), `/tmp/BdsConfiguration.xml` 27 s, `/tmp/vui_emnavi_poi.eco` 48 s, `/tmp/db1_con_cache.sqlite` (400 KB) and `/tmp/db1_ch_cache.sqlite` 54 s, `/tmp/vui_contacts0.eco`, `/tmp/sms.db`, `/tmp/email.db` 57 s. All after ~45 s are later than HomeScreen and the Android Auto projection.
- RACE (USB media) keeps its resume DB only in tmpfs (`/mnt/tmp/race/database/RACE_ResumePlay.db`); at boot the restore finds nothing (`Stat failed`) and costs no time.
- Fail-safe version: `jciUpdatea` runs `jci-failsafe-version.sh` (31.3 → 34.0 s in the log). Its fast path reads 22 bytes at offset 7339776 of the `fail-safe` MTD (`/dev/mtd7`, `MAZ_CMU-150_74.00.324A`); that `dd` takes ~0.03 s and the MTD name lookup ~0.07 s on an idle system, so the 2.7 s is scheduling delay under boot load.
- Opera maps every font in `OPERA_FONTS` (`/jci/fonts/CHN`, 17 TTF files incl. CJK, Arabic, Hebrew, Thai) at start-up. Opera's disk cache is off (`[Disk Cache] Enabled=0`); Presto keeps no compiled-JS cache on disk.
- No `/etc/ld.so.cache` and no fontconfig on the rootfs.
- SiriusXM services `jciSXMS` and `jciXMMGR` are `autorun="yes"` in both SM configs, also on EU units; `XM_RADIO` logs tuner status at ~22 s.

## Media album art sources and the Gracenote dependency (fw 74.00.324A, stock image strings)

- **USB media** (`/jci/usbm/usbms` + `/usr/lib/librace_interface.so`, RACE): album art is read from the file itself first (`race_get_albumart_info` / `race_read_albumart`: offset, size, MIME type of the embedded picture). Only when that fails does it fall back to Gracenote (`Failed to retrieve Albumart from device, trying to get Gracenote Albumart` → `GRACENOTE_CoverArtLookup`). The art is saved as a BMP under `/tmp/albumart` and announced with the `com.jci.usbms` `AlbumArtReady` signal.
- RACE uses Gracenote for: fallback cover art, metadata lookup (`GRACENOTE_DBLookup`), "More Like This" playlists (`race_gnw_get_morelikethis`), and a collection DB (`/tmp/gracenotedb/`, saved to `/data/gracenotedb_save/`). It has failure paths (`Gracenote support is disabled`, `Can not connect to gracenote server.`, `Gracenote wrapper init failed`); no reconnect logic is visible in the strings, so a Gracenote that is absent at RACE init most likely stays unavailable for that boot.
- `jciUSBMS` and `jciUSBM` have no `gracenote_server` dependency in `sm.conf` (only `devicemanager`, and `jciUSBMS`/`jciBLMSettings`/`audio_manager`/`jciRM`). `jciBTMUSIC` does depend on it.
- **Bluetooth media** (`bds`, `svcjcibtmusic.so`): no AVRCP 1.6 / BIP cover-art strings; BT art comes only from Gracenote (`No gracenote update. No album art.` otherwise).
- **Observed on the bench (USB stick, FAT32):**
  - Embedded ID3v2 `APIC`, baseline JPEG 160×160: shown.
  - Embedded `APIC`, progressive JPEG 457×639: not shown. RACE logs `race_dbm_get_albumart_info ... Marker FFC2 found`, then `error=0x313`, so progressive JPEG is not supported.
  - `Folder.jpg` / `AlbumArt*.jpg` next to the tracks, nothing embedded: not shown. RACE reads only the embedded picture.
  - No picture at all: not shown.
  - Each failure then tries the Gracenote fallback (`race_gnw_get_albumart`).
- **Gracenote started late:**
  - When `gracenote_server` starts after the boot-time `usbms` (~21 s), the RACE in that process logs `Gracenote wrapper init failed. Error=0x800`. Every later fallback then fails with `race_gnw_get_albumart ... Recieved NULL Pointers`. Playback uses this process.
  - A stick inserted later starts a separate indexing process, and that one connects to Gracenote normally.
  - BTMUSIC that starts without Gracenote logs `ServiceInit) Gracenote Connect failed` and does not retry; BT tracks get title and artist but no art until the next boot.

## Android Auto entry in Applications becomes active (fw 74.00.324A)

- The Android Auto item in the Applications menu turns active on `MMUI_ANDROIDAUTO ... uiaAndroidauto_Available_cb) [Androidauto]Available Status true`, sent right after `aap_create_session returns 0` (`BLM_AAPA ... AAPA_Available signal send`). This happens before projection starts, and does not need it.
- The status-bar icon (`UIA_SYSTEM_OnStatusMenuUpdate appName=androidauto ... appStatus=1`) comes much later (~44–55 s), together with projection.
- One cold power-on, stopwatch from the power button to the item turning active: 41 s, log 34.4 s, so ~6.6 s between the button and the log's time base plus GUI render.

## Critical chain to the boot animation and Home (fw 74.00.324A, one boot, SM tweaks: `mfid` on `stage_3`, GUI framework holds background apps)
`sm.conf` + SM events + `running_log`, uptime in s:

| t | event |
|---|---|
| 13.72 | `Opera` started |
| 15.22 → 17.48 | `audio_control` (`audioCtrl.lua`) start → started (2.3 s) |
| 17.48 → 17.89 | `audio_manager` |
| 17.90 → 18.97 | `jciRadio`, `jciCD`, `jciauxin` (depend on `jciVBS`, `audio_manager`; `jciRadio` also `settings`, `jciTds`) |
| 19.03 | `jciMMUI` start (depends on `jciBUCPSA`, `jciBLMSettings`, `jciRadio`, `jciauxin`, `jciCD`) |
| 21.07 | GUI: "GUI is ready. Sending Global.GetStartupSettings event to MMUI." |
| 23.42 | `jciMMUI` started (3.9 s after start) |
| 24.09 | MMUI processes `GetStartupSettings` (its event-queue threads wait for a "run semaphore" until MMUI is started) |
| 24.34 | `tranlogo is started` / `IntroAnimation` |
| 29.0 – 29.3 | `tranlogo` stopped, `Disclaimer` context |
| 30.2 | `DisposeIntroVideo` (right after the `vehsettingsApp` "not initialized" line) |
| 30.7 | `HomeScreen` |

- Across six boots the GUI was still initialising for 1.2–2.9 s after `tranlogo` ended; `HomeScreen`
  followed `DisposeIntroVideo` by 0.4–0.7 s. During that gap the last `TranLogo` frame (a picture of
  the Home menu without the status bar) stays on screen.
- So the GUI waits about 3 s for MMUI, and MMUI waits for the audio chain
  (`audio_control` → `audio_manager` → tuner/CD/AUX services).
- All metric markers used for boot timing (`tranlogo is started`, `ctxtChg`, `DisposeIntroVideo`,
  `[Androidauto]Available Status true`) are Info-level MMUI/GUICORE lines.
- Inside `jciMMUI`: `svcjcimmui.so` (23 KB) plus `libjciuiamgr.so` (UIAMGR) load the UIA modules listed in
  `/jci/mmui/mmui_config.xml`, in file order, all `block="true"`: each library is loaded and its
  `UIA_<NAME>_Open` is called before the next one (`failaction="continue"`). 43 modules: common, system, backupparking, emnavi, btpairing, netmgmt, email,
  contacts, pandora, usbaudio, phone, fmradio, sysupdate, btaudio, audiosettings, amradio, mobile911, sms,
  syssettings, vehsettings, auxin, satradio, diag, cd, dvd, tv, stitcher, aharadio, favorites,
  hdtrafficimage, ecoenergy, schedmaint, warnguide, tutorial, driverid, idm, dab, vdt, vsm, carplay,
  androidauto, xmaudio, xmdata.
- Same boot: `jciMMUI` start 19.03, first MMUI log line 20.16, UIA module log lines appear one after
  another from 20.8 (`MMUI_SYSTEM`) to 23.5 (`MMUI_DRIVERID`), about 70 ms per module; SM `STARTED` 23.42.

### jciMMUI start-up internals (from the stock binaries)

- `ServiceInit` in `svcjcimmui.so` runs the whole start-up on one thread: parse the SM args
  (`mouse=`, `keyboard=`, `display=`, WCP board flag), `UIAM_New` (opens the GUI link, starts the TUI
  input reader threads and the two UIAMGR event-queue threads), `mmuiLoadUias`, then
  `GUIIFM_Internal_AppsLoadingFinish` and `UIAM_Run`. `ServiceRun` is only a 1 s SM ping loop.
- The two UIAMGR threads (`blockedEventQueueThread`, `nonblockedEventQueueThread`) wait on a "run
  semaphore". Only `UIAM_Run` posts it, after every UIA module is loaded. Until then no GUI event is
  processed.
- `mmuiLoadUias` reads `/jci/mmui/mmui_config.xml`, then for each `<Config>` calls `dlopen(path,
  RTLD_NOW)` and the `api` function (default `APPL_Main`) synchronously. `svcjcimmui.so` imports no
  `pthread_create`: the `block` attribute does not create a thread, despite the XML header comment.
  It only selects whether the return value is compared with `passvalue` (mismatch → `exit(1)`, and
  `jciMMUI` has `reset_board="yes"`).
- `lib="ALL"` loads every `.so` in `libpath` (directory scan); the per-module form is used in stock.
- The UIA directory `/jci/mmui/uia/` holds 44 libraries, about 11 MB in total (largest: xmaudio
  0.89 MB, vehsettings 0.76 MB, syssettings 0.71 MB, system 0.67 MB).
- Effect on Home: the GUI logs `GUI is ready. Sending Global.GetStartupSettings event to MMUI` at
  20.8–21.0 s, but `GetStartupSettings` is processed only at `UIAM_Run` (23.7 s) in three measured boots.
  The intro animation and Home follow that reply.
- Per-module log windows in one boot (first log line to next module's first line): TUI/common
  0.60 s, backupparking 0.21, pandora 0.24, audioaggregator 0.19, amradio 0.18, syssettings 0.18;
  most others 0.02–0.13 s.
- Radio, CD and AUX modules do not need their services to be up when they open: each queries SM
  (`SM_GetServices`), uses the service when it reports "already running", and otherwise registers an
  SM started-callback (`sm_event_fm_started_cb`, `sm_event_am_started_cb`, `sm_event_aux_started_cb`,
  CD `SM_EVENT_SERVICE_STARTED`). `jciMMUI` in `/jci/sm/sm.conf` still depends on `jciBUCPSA`,
  `jciBLMSettings`, `jciRadio`, `jciauxin`, `jciCD`; `tranlogo` and `quicktranlogo` depend on `jciMMUI`.
