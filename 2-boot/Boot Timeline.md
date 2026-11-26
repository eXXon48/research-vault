---
tags: [cmu150, boot, sm, timing]
---

# Boot Timeline

When the System Manager (SM) stages and the main services start in a normal boot, what each
stage waits on, and the dependency chains that set the times of the Home screen and Android Auto.

## Overview

A normal boot has three phases:

| Phase | Time |
|---|---|
| Kernel start to SM start | ≈ 9 s |
| SM start to the HMI stage (`stage_3`) | ≈ 15.6 s |
| SM start to SM `RUNNING` (all autorun services up) | ≈ 49.5 s |

The SM phase dominates. Most of it is service start-up in series behind `jciMMUI` and the stage
barriers, with core 0 saturated from 8 s to about 68 s.

- **Home screen** (~31–42 s, depending on tweaks): the GUI waits for `jciMMUI`, which waits for the
  audio chain and loads 43 modules one after another. The boot animation then sets the earliest
  Disclaimer time.
- **Android Auto**: waits on the Bluetooth chain `bds` → `jciPA` → `jciBCA` → `mfid` →
  `devicemanager` → `jciAAPA`.

Tested changes and their effect: [[Boot Speed Levers]].

Related: [[Boot Chain and Fail-Safe]] (before the kernel and the initramfs),
[[SM Service Graph]], [[SM Service Control (smctl)]], [[FlashFX Read Path]], [[System Clock]].

Times are seconds since kernel start unless noted. Time before the kernel (boot ROM, bootstrap,
IBC) is not visible in these clocks.

## Reference

### Milestones

One boot. Sources: the SM event dump, the kernel log of the same boot, stock `sm.conf`.

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
| 58.53 | SM `RUNNING` |

Event counts in that boot: 107 `SM_SVC_CMD_START`, 112 `STARTED`, 64 handshakes.

Longest start → started: `bds` 8.3 s · `jciCARPLAY` 7.2 s · `start_network` 6.6 s · `stage_vr`
5.2 s (fixed sleep) · `jciVBS` 5.2 s · `jciVwm` 5.0 s · `jcinavi` 4.8 s · `jciblmVdt` 4.4 s ·
`jciMMUI` 4.3 s · `jciAAPA` 3.9 s · `jciUSBM` 3.9 s.

### Variation between boots

Five consecutive SafeReboot (warm) boots, legacy board, `sm.conf`, each left up for more than
150 s. The unit had third-party packages installed. Spread = max − min.

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

The spread is under 1 s up to `stage_3`, and about 2 s after it (`start_network` up to 3.9 s). A
single change has to move a milestone by more than that to show in five boots.

### Stage graph (`sm.conf`)

- 117 `<service>` entries.
  - 91 carry `reset_board="yes"`, including `Opera`, `jciMMUI`, `bds`, `jciUSBMGR`,
    `jciCARPLAY`, `jciAAPA` and the audio chain.
  - 26 carry `reset_board="no"`, including all `stage_*`, `emmc_mount`, `usb_drivers` and
    `start_network`.
- The `<services>` default is `affinity_mask="0x01"` (cpu0); 18 services set `0x02`.

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

- `jciMMUI` depends on `jciBUCPSA`, `jciBLMSettings`, `jciRadio`, `jciauxin` and `jciCD`, so the
  tuner and CD services sit on the HMI critical path. `tranlogo` and `quicktranlogo` depend on
  `jciMMUI`.
- `Opera` depends on `jciAppsdk` and `jciDbapi`.
- `stage_vr.sh` is `sleep "$1"`, then `touch /var/run/stage_vr_ready`. With `args="5s"` it adds a
  fixed 5 s before the VR-gated services.
- `stage_wifi`, `stage_gap2` and `stage_vr` all wait for `blm_ira_aha` (Aha streaming client).
  `blm_ira_aha` itself depends on `stage_ir`, `offboard_service`, `bds` and `audio_manager`.
- The resources partition (`mmcblk0p1`) is mounted only after `stage_3`, by SM service
  `emmc_mount` (`/usr/sbin/emmc_mount.sh`). `/data_persist` (`mmcblk0p2`) is mounted before SM
  starts.
- Leaves (no dependents): `jcidvd`, `jcitv`, `jciDAB`, `jciXMMGR`, `blm_ira_pandora`,
  `blm_ira_stitcher`. `jciSXMS` has only XM services as dependents. `jciCD` is not a leaf:
  `jciMMUI` depends on it. A leaf can still delay the boot when dropped; see
  [[Boot Speed Levers]], "DVD and TV services off".
- SiriusXM services `jciSXMS` and `jciXMMGR` are `autorun="yes"` in both SM configurations, EU
  units included. `XM_RADIO` logs tuner status at ~22 s.

### SM configuration and the event file

`/usr/bin/autostart` (normal-mode branch) does the following:

1. Sources `/jci/scripts/start_normal_mode.sh`.
2. Starts the logger.
3. `sleep 1`.
4. Runs `/jci/scripts/get_board_type.sh`.
5. Starts SM: `taskset 0x02 /jci/sm/sm -f /jci/sm/sm.conf -e /tmp/smevents.txt`.

`get_board_type.sh` decides between `sm.conf` and `sm_WCP.conf` ([[Hardware and Firmware]],
"Board variants"). On a legacy board, SM runs with `-f /jci/sm/sm.conf`. A change to SM services
has to be made in both files to cover both boards.

The event dump is the `-e` file, `/tmp/smevents.txt`: a CSV `PID,Component,Event,Timestamp`,
timestamps in seconds since kernel start. It is written fresh every boot on tmpfs and lost at the
next reboot. To keep a timing record, copy it off after each boot. Event sequence per service and
how to detect "SM has finished starting": [[System Clock]], "Detecting that SM has finished
starting".

### Critical chain to `RUNNING`

For each service, the gate is the dependency that finished starting last. Followed back from the
last service to start (`jciMSG`), one boot:

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

`stage_vr` is the only pure wait on the chain, and its only dependent is `natp`. Its 5 s add in full
to everything after it.

A second capture (third-party packages installed) shows the Wi-Fi/CarPlay chain, each step waiting
on the one before:

`bds` started 34.15 · `jciBCA` 35.84 → 37.00 · `stage_ir` 37.06 · `blm_ira_aha` 37.25 → 40.34
(3.1 s) · `stage_wifi`/`stage_gap2` 40.37 · `jciCARPLAY` 42.42 → 47.16 · `stage_vr` 52.37.

In the same window, `irsstitcher_server` (37.19 → 37.98), `blm_ira_stitcher` (38.00 → 39.92) and
`blm_ira_pandora` (38.03 → 40.21) start on core 0 alongside `blm_ira_aha`.

### Critical chain to the Home screen

One boot with SM and GUI tweaks (`mfid` on `stage_3`; the GUI framework holds background apps).
`sm.conf` + SM events + `running_log`:

| t | Event |
|---|---|
| 13.72 | `Opera` started |
| 15.22 → 17.48 | `audio_control` (`audioCtrl.lua`) start → started (2.3 s) |
| 17.48 → 17.89 | `audio_manager` |
| 17.90 → 18.97 | `jciRadio`, `jciCD`, `jciauxin` (depend on `jciVBS`, `audio_manager`; `jciRadio` also on `settings`, `jciTds`) |
| 19.03 | `jciMMUI` start |
| 21.07 | GUI: "GUI is ready. Sending Global.GetStartupSettings event to MMUI." |
| 23.42 | `jciMMUI` started (3.9 s after start) |
| 24.09 | MMUI processes `GetStartupSettings` |
| 24.34 | `tranlogo is started` / `IntroAnimation` |
| 29.0 – 29.3 | `tranlogo` stopped, `Disclaimer` context |
| 30.2 | `DisposeIntroVideo` (right after the `vehsettingsApp` "not initialized" line) |
| 30.7 | `HomeScreen` |

- The GUI waits about 3 s for MMUI. MMUI waits for the audio chain: `audio_control` →
  `audio_manager` → tuner/CD/AUX services.
- Across six boots, the GUI was still initialising for 1.2–2.9 s after `tranlogo` ended.
  `HomeScreen` followed `DisposeIntroVideo` by 0.4–0.7 s. During that gap the last `TranLogo` frame
  (a picture of the Home menu without the status bar) stays on screen.
- All boot timing markers (`tranlogo is started`, `ctxtChg`, `DisposeIntroVideo`,
  `[Androidauto]Available Status true`) are Info-level MMUI/GUICORE lines.

### Inside `jciMMUI` start-up

From the stock binaries:

- `ServiceInit` in `svcjcimmui.so` (23 KB) runs the whole start-up on one thread:
  1. Parse the SM args (`mouse=`, `keyboard=`, `display=`, WCP board flag).
  2. `UIAM_New`: opens the GUI link and starts the TUI input reader threads and the two UIAMGR
     event-queue threads.
  3. `mmuiLoadUias`.
  4. `GUIIFM_Internal_AppsLoadingFinish` and `UIAM_Run`.

  `ServiceRun` is only a 1 s SM ping loop.
- The two UIAMGR threads (`blockedEventQueueThread`, `nonblockedEventQueueThread`) wait on a "run
  semaphore". Only `UIAM_Run` posts it, after every UIA module is loaded. Until then no GUI event is
  processed.
- `mmuiLoadUias` (with `libjciuiamgr.so`, UIAMGR) reads `/jci/mmui/mmui_config.xml`. For each
  `<Config>`, in file order, it calls `dlopen(path, RTLD_NOW)` and then the `api` function (default
  `APPL_Main`, the `UIA_<NAME>_Open` call) synchronously. `failaction="continue"`.
- `svcjcimmui.so` imports no `pthread_create`. Despite the XML header comment, the
  `block="true"` attribute does not create a thread. It only selects whether the return value is
  compared with `passvalue`. A mismatch calls `exit(1)`, and `jciMMUI` has `reset_board="yes"`.
- `lib="ALL"` loads every `.so` in `libpath` (directory scan). Stock uses the per-module form.
- 43 modules, in order: common, system, backupparking, emnavi, btpairing, netmgmt, email, contacts,
  pandora, usbaudio, phone, fmradio, sysupdate, btaudio, audiosettings, amradio, mobile911, sms,
  syssettings, vehsettings, auxin, satradio, diag, cd, dvd, tv, stitcher, aharadio, favorites,
  hdtrafficimage, ecoenergy, schedmaint, warnguide, tutorial, driverid, idm, dab, vdt, vsm, carplay,
  androidauto, xmaudio, xmdata.
- `/jci/mmui/uia/` holds 44 libraries, about 11 MB in total. Largest: xmaudio 0.89 MB,
  vehsettings 0.76 MB, syssettings 0.71 MB, system 0.67 MB.
- Radio, CD and AUX modules do not need their services when they open. Each queries SM
  (`SM_GetServices`) and uses the service when it reports "already running". Otherwise it registers
  an SM started-callback: `sm_event_fm_started_cb`, `sm_event_am_started_cb`,
  `sm_event_aux_started_cb`, CD `SM_EVENT_SERVICE_STARTED`.

Timing, same boot as the table above:

- `jciMMUI` start 19.03, first MMUI log line 20.16.
- UIA module log lines appear one after another from 20.8 (`MMUI_SYSTEM`) to 23.5
  (`MMUI_DRIVERID`), about 70 ms per module. SM `STARTED` 23.42.
- Per-module windows (first log line to next module's first line): TUI/common 0.60 s,
  backupparking 0.21, pandora 0.24, audioaggregator 0.19, amradio 0.18, syssettings 0.18; most
  others 0.02–0.13 s.
- In three boots, the GUI logs `GUI is ready. Sending Global.GetStartupSettings event to MMUI` at
  20.8–21.0 s, but MMUI processes `GetStartupSettings` only at `UIAM_Run` (23.7 s). The intro
  animation and Home follow that reply.

### Screens during boot

MMUI sends each screen change to the GUI as `{"msgType":"ctxtChg","uiaId":…,"ctxtId":…}`, logged by
`GUICORE … GUIIFM_SendDataToGUI` in `/var/log/running_log/log`. One stock boot (uptime from the
log's wall clock, with the offset from `date` + `/proc/uptime` read together):

| t (s) | Screen |
|---|---|
| 24.89 | `system` / `IntroAnimation` (boot screen) |
| 30.37 | `system` / `Disclaimer` |
| 41.86 | `system` / `HomeScreen` (after `UIA_SYSTEM_Event_Timeout` from the GUI) |

**Boot animation.**

- MMUI starts SM service `tranlogo` on demand, right before it sends `IntroAnimation`
  (`UIA_SYSTEM_GetStartUpSettings` "tranlogo is started").
- `tranlogo` is
  `/usr/bin/splashplayer -q -p /var/run/tranlogo_ready_flag -t TRANLOGO_SURFACE -z 700 /jci/resources/TranLogo.ivf`
  (`sm.conf`, `autorun="no"`, cpu1). `TranLogo.ivf` is 1 240 681 bytes.
- `TranLogo` is the Mazda logo turning into a pre-rendered picture of the Home menu. Its last
  frames are video, not the live GUI.
- Three boots (stock, and two with GUI tweaks): `tranlogo` runs 4.4–4.8 s (start 25.46–26.76 →
  `Svc tranlogo stopped!` 30.07–31.54). MMUI sends `ctxtChg Disclaimer` 0.6–1.9 s after that
  stop, in every boot. So the boot animation, not the GUI, sets the earliest Disclaimer time.
- After the Disclaimer, MMUI stops `quicktranlogo` (`LoopLogo.ivf`, looped), which is already
  stopped. NATGUICTRL keeps showing the `Newlooplogo` surface until the GUI takes over.

| IVF file | Frames |
|---|---|
| `TranLogo.ivf` | 120 |
| `TranLogoEnd.ivf` | 30 |
| `LoopLogo.ivf` | 54 |
| `newLoopLogo.ivf` | 60 (30 fps header) |

All are VP8, 800×480, 90 000 timebase. `splashplayer` options:

| Option | Meaning |
|---|---|
| `-r <fps>` | frame rate, default 30 (so `TranLogo` lasts 120/30 = 4.0 s plus start-up) |
| `-l` | loop |
| `-q` | quit at end |
| `-p` | pid file |
| `-t` | surface title |
| `-x -y -w -h -z` | window position, size and z-order |
| `-f` | format (9 = VP8) |

**Disclaimer.**

- `systemApp.js` (`_disclaimerTime`) starts a 3500 ms timer when the Disclaimer shows, and sends
  `Timeout` to MMUI when it fires. The timer pauses while another screen covers the Disclaimer.
  `Global.Yes` on the Disclaimer sends the same exit early.
- The Disclaimer's displayed callback sends `DisposeIntroVideo` and starts the timer. It runs only
  once the GUI has finished initialising its apps. In five boots `DisposeIntroVideo` came:
  - 0.0–0.5 s after the `vehsettingsApp.js` "context/template not initialized" error line;
  - 5.8–8.9 s after the Disclaimer context.
- `DisposeIntroVideo` is when NATGUICTRL switches from the native `Newlooplogo` surface to
  `JCI_OPERA_PRIMARY`. The boot animation therefore lasts until the GUI is ready.
- In the stock boot above, `Timeout` reached MMUI 11.5 s after the Disclaimer context was sent, not
  3.5 s. Unmoved boots show a Disclaimer of 9.3–10.0 s.

**Which loop logo plays: `/data/unexpectedshutdownflag`.**

`/sbin/init_cmu` reads `/data/unexpectedshutdownflag` (one byte) before SM starts, and picks the
loop logo from it:

| Flag | init log | Loop logo | What is seen |
|---|---|---|---|
| `1` | `<I> Detected CMU reboot in previous cycle` | `splashplayer -l -t NEWLOOPLOGO_SURFACE -z 1700 -p /var/run/newlooplogo.pid /jci/resources/newLoopLogo.ivf` | logo + spinner until the Home screen; the `tranlogo` animation (z 700) and the Disclaimer still run but are covered |
| other | — | `splashplayer -l -t LOOPLOGO_SURFACE -z 500 -p /var/run/looplogo.pid /jci/resources/LoopLogo.ivf` | loop logo, then the `tranlogo` animation and the Disclaimer on top |

- Reader: `cb_looplogo` in `/sbin/init_cmu` (also `init_cmu_rr`) reads one byte. Any byte other
  than `'0'` selects the newlooplogo path; a missing file does not. No other binary reads the file.
- Writer: `SYSTEM_SMM_UnexpectedShutdown(int)` in `/jci/system/svcjciblmsystem.so` writes `%d` +
  `sync()`, only when the value differs from the file.

| Caller | Writes | When |
|---|---|---|
| `system_smm_PrepareForState`, target `NormalState` (2) | `1` | every boot, when the system reaches Normal |
| `system_smm_PrepareForState`, target `StandbyState` (5) | `0` | orderly ACC off (display ACC-off or sleep, audio off) |
| `DbusServerClientModule::genericReboot` (D-Bus `GenericReboot`) | `0` | after `SYSTEM_HSRM_StartBoardReset`/`StartShutdown` returns 100 |
| `factoryReset`, `system_smm_VIPReflash_EventHandler`, `SYSTEM_SMM_EnterVIPReflashMode` | `0` | factory reset, VIP reflash |

- SMM state enum (`SystemStateMachine` name table): 0 `InvalidState`, 1 `SYSTEM_InitState`,
  2 `SYSTEM_NormalState`, 3 `SYSTEM_IdleState`, 4 `SYSTEM_DelayState`, 5 `SYSTEM_StandbyState`,
  6 `SYSTEM_PoweringDownState`, 7 `SYSTEM_TestState`, 8 `SYSTEM_VipReflashState`,
  9 `SYSTEM_MAP_SleepState`, 10 `SYSTEM_MAP_ColdStartState`.
- `DbusServerClientModule::safeReboot` (D-Bus `SafeReboot`) never writes the flag. It calls
  `SetEmergency`, `SetDisplayOff`, `SetAudioOff`, `SYSTEM_HSRM_StartBoardReset(0,1,0)`. The `1`
  from Normal stays, and the next boot shows newlooplogo. A power cut also leaves `1`. Only an
  orderly ACC off (Standby) gives the LoopLogo path.
- `GenericReboot` with `mode_flags` = 0 decodes to the same `StartBoardReset(0,1,0)` with display
  and audio off, plus the `0` write. Flag bits:

  | Bit | Meaning |
  |---|---|
  | `1` | keep the display on (also passed to `StartBoardReset`) |
  | `2` | clears argument 2 |
  | `4` | shutdown instead of reset (refused unless bit 2 is set and bit 1 is clear) |
  | `8` | argument 3 |

- In the newlooplogo path, `jciNativegui` shows the surface again on idle
  (`NGC_CheckNewLooplogoAnimation`, `OnIdle: Showing Newlooplogo surface...`). MMUI stops it when
  the Home screen comes (`uiaSystem_Stop_NewLoopLogo`: `kill -TERM $(cat /var/run/newlooplogo.pid)`).
  On this path a shorter `tranlogo` cannot be seen. It matters only by when MMUI sends the
  Disclaimer.
- The flag changes only what is seen, not the timing. Stock, `GenericReboot int32:0`: log
  `read /data/unexpectedshutdownflag value is [0]`, LoopLogo path, Disclaimer context 33.20 s,
  `DisposeIntroVideo` 33.56 s, Home 37.06 s. The newlooplogo path on the same unit gave Home 37.2 s.

### Critical chain to Android Auto

Two boots. b40: phone plugged in during the boot, GUI tweaks (Home 30.6 s). b43: stock, no phone;
its SM chain is the same within 0.5 s.

| Step | b40 | b43 |
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

- Android Auto waits on the Bluetooth stack. `bds` takes 7.7 s from start to ready flag, ~1.4 s of
  it the BlueGo UART init. Then `jciPA` → `jciBCA` → `mfid` → `devicemanager` (4.5 s) → `jciAAPA`.
- The USB hubs are up at ~14.6 s (`jciUSBMGR` started). The phone is enumerated long before the
  switch to accessory mode.
- The chain is slow from waiting, not from its own work: `bds` used under 0.6 s of CPU (utime +
  stime) in its 7.7 s start. Both cores are 90–100 % busy from ~25 s to ~50 s.
- The first `BDS` line in the running log comes ~3 s after SM starts `bds`.
- From `jciAAPA` start to the Android Auto screen: ~7.8 s. That is the AOA switch (~0.5 s), the
  AAP session create (~3 s) and the Bluetooth pairing check (~1.8 s).
- AAP session create (~2.9 s, `Calling service create session` → `Session created`): XML config
  and certificates ~0.3 s, then RACE init (Gracenote client, DB restore), and a
  `gst-plugin-scanner` run (GStreamer 0.10 registry rebuild, `Failed to load plugin` warnings) in
  the last ~1.5 s. The scanner runs on every stock boot ([[Boot Speed Levers]], "GStreamer
  registry").
- `devicemanager` (`/jci/devicemanager/libdevicemanager.so`) handles both iAP (CarPlay, iPod) and
  AOA (Android Auto) devices. It logs nothing to the running log during its start; its own log
  targets are `/tmp/RACE_LOG.txt` and `/tmp/devmgr_debug.txt`.
- `mfid` (`/usr/bin/mfid`) is the Apple MFi authentication-chip daemon: I2C (`/dev/i2c*`,
  `Device I2C address`), unix socket `/tmp/MFISocket`. It links only `libprotobuf-c`, `librt`,
  `libpthread` and `libc`, so it uses no Bluetooth or D-Bus. Its `jciBCA` dependency in `sm.conf`
  and `sm_WCP.conf` is ordering only. Users of the MFi socket/library: `blmjcicarplay.so`,
  `ipoddev_full_auth`, `libmfi.so`, `libCarPlayPlatformMFI.so`.
- Services downstream of `mfid` (transitive `sm.conf` dependencies): `devicemanager` →
  `aap_service`, `jciCDRP`, `jciUSBMS` (→ `jciUSBM`, `jciITTS` → `jciITT`), `jciCARPLAY`
  (→ `stage_vr` → `natp` → `jciTTSPlayer`, `jciVUI` → `kapture`; → `jcibteca` → `jciPB` →
  `jciMSG`), `jciAAPA`.
- `jciAAPA` also needs `jciRM` (~25.5 s) and `jciUpdatea` (~28.8 s).

**Android Auto entry in Applications.** The Android Auto item in the Applications menu turns
active on `MMUI_ANDROIDAUTO ... uiaAndroidauto_Available_cb) [Androidauto]Available Status true`.
That line comes right after `aap_create_session returns 0` (`BLM_AAPA ... AAPA_Available signal
send`), before projection starts; projection is not needed. The status-bar icon
(`UIA_SYSTEM_OnStatusMenuUpdate appName=androidauto ... appStatus=1`) comes much later
(~44–55 s), together with projection.

### Wireless Android Auto adapter

Adapter: USB name `smartBox`, serial `LY0123456789`. It presents as an Android device and is
switched to AOA (`18d1:2d00`) by `CPP_USBMGR`. "AOA dev" = re-enumerated in accessory mode,
"created" = `Service:Session created`.

| Boot | AOA dev | AAP `session start` | created | Adapter detach | Projection |
|---|---|---|---|---|---|
| b50 | 36.6 | 37.2 | 41.9 | — | 42.2 |
| b53 | 36.2 | 37.7 | 41.5 | — | 42.1 |
| b49 | 23.0 → detach 31.6, again 43.3 | 43.3 | 45.9 | 31.6 | 46.4 |
| b47 | 36.7 | 43.0 | 45.3 | 46.3 | 57.7 |
| b51 | 32.3 | 36.7 | 40.9 | 41.5 | 55.2 |
| b52 | 32.6 | 37.7 | 42.4 | 41.8 | 55.6 |

- In accessory mode the adapter waits ~8.6–9.8 s for the head unit's AAP traffic. Then it
  detaches and stays off the bus ~10–13 s. One boot (b48) kept it for ~23 s.
- `jciAAPA` picks up an adapter that attached before it started
  (`FindAndroidAutoDevice: AndroidAuto device is already connected` → `StartSession`).
- The time is lost in the first `session start` → `Session created` step: 2.3–4.7 s, in the
  CPU-heavy window, including the per-boot `gst-plugin-scanner` run.
- Once a session exists, a re-attached adapter reaches projection ~0.5 s after `session start`.
- Projection comes on the first try when `Session created` lands within ~8 s of the adapter's AOA
  re-enumeration. Otherwise the adapter costs one full reconnect (~11 s). Example b46 (Home
  32.4 s): accessory mode at ~32.8 s, `jciAAPA` up only at 41.5 s; the adapter detached at 41.45 s,
  accessory mode was set again at ~53.0 s, `AAP_PROJECTION_SETUP` at ~54.4 s.
- Whether the drop is the adapter's own timeout or a phone-link event is not known. A phone on a
  USB cable may behave differently.

### `natp`

`/jci/natp/bin/natp -f /jci/natp/config/natp.cfg -s natp_config -k VerboseLevel=0` is the Nuance
speech platform.

- `natp.cfg` channels: HostAgent, MonitorService, GDM, AudioMatrixService,
  AudioRecordingService, PrompterService, SAIP/SAOP/SAS, FileService, VoconService (Vocon ASR, TTS
  decoder).
- In `sm.conf` it depends on `emmc_mount` and `stage_vr`, and carries `reset_board="yes"`.
  `jciTTSPlayer` and the voice UI depend on it.
- It starts after HomeScreen (≈45–51 s).

### Fixed waits in boot-path scripts

From the stock rootfs (`usr/bin/autostart` and the scripts named in `sm.conf`):

| Location | Wait |
|---|---|
| `usr/bin/autostart:78` | `sleep 1` before `get_board_type.sh`, so before SM starts (normal mode) |
| `jci/scripts/stage_vr.sh:2` | `sleep "$1"` (`args="5s"`) |
| `jci/scripts/jci-wifiap.sh` | `sleep 2` at lines 62, 76, 92, 105 (different code paths); `sleep 1` at 185 |
| `jci/scripts/get_board_type.sh` | `sleep 1` only on a failed GPIO read (retry loop) |
| `usr/sbin/emmc_mount.sh:10` | `waitfor /sys/block/${MMC_DEV}/dev 30000` (bounded poll, not a fixed wait) |
| `jci/scripts/wait_core_finished.sh` | 1 s poll loop on `/var/lock/*.core`, up to 300 s |

Timers inside binaries are not visible here. For example, BCA logs
`Incoming Connection Wait timer started with timeout 5000 milli sec` twice around 35.9–36.0 s.

### CPU and I/O during boot

**Sampled from `/proc`, one boot.** Method: a BusyBox shell loop started from the top of
`start_normal_mode.sh` (6.6 s), pinned to cpu1. Every ~2.4 s it appends the `/proc/stat` cpu lines,
`/proc/diskstats` (`ffx01`, `mmcblk0`) and each `/proc/<pid>/{stat,statm,io}`. One sample costs
~0.4 s CPU (≈190 processes). The sampler itself used 5.9 s of cpu1 up to 59 s and delayed
`stage_3`/`RUNNING` by ~1 s. Processes born and gone between two samples are not seen.

- **cpu0 is saturated from 8 s to 68 s** (88–100 % busy, iowait ≤ 13 %). cpu1 runs 20–60 % (about
  20 % of it the sampler), except at 28–40 s, when both cores are at 95–100 %. After ~70 s, cpu0
  settles at 25–40 % and cpu1 at ~20 %.
- Rootfs reads (`ffx01`) run at 2–6 MB/s from 8 s to ~58 s. `mmcblk0` reads burst at 56–66 s (up
  to ~5.8 MB/s).
- **`ffxread01`**, the kernel thread serving `ffx01` reads, is the largest consumer
  ([[FlashFX Read Path]]): 5.3 cpu-s up to `stage_3` (4.9 s of it on cpu0), 10.5 cpu-s up to 59 s.
  Its affinity is `0-1` (`Cpus_allowed_list`, not `PF_THREAD_BOUND`), but it is scheduled almost
  only on cpu0.

| cpu-s used | 6.6 s → 25.4 s (`stage_3`) | 6.6 s → 58.8 s |
|---|---|---|
| `ffxread01` | 5.34 | 10.45 |
| `opera` | 1.87 (cpu1) | 6.53 |
| `jciBLMSetting` | 1.45 (cpu1) | 3.50 |
| `jciMMUI` | 1.16 | 2.43 |
| `syslog-ng` | 0.94 | 2.55 |
| `jciVBS` | 0.94 | 2.49 |
| SM main thread | 0.63 (cpu1) | 1.64 |
| `vim_app` | 0.61 | 2.18 |
| `settings` | 0.57 | 1.44 |
| `splashplayer` | 0.52 (cpu1) | 1.45 |
| `dbus-daemon` | 0.49 + 0.35 | 1.52 + 1.00 |
| `wayland-viv-compositor` | 0.46 (cpu1) | 1.21 |
| `jciUSBMGR` | 0.42 | |
| `natp` | | 1.19 |
| `jciCARPLAY` | | 1.18 |
| `LVDS_MAIN` | | 0.92 |
| two `lua` | | 0.91 + 0.84 |
| `process-log` | | 0.79 |
| `bds` | | 0.71 |

The sampler excluded. `RUNNING` came at ≈ 59.7 s in this boot. Of 71.5 cpu-s seen up to 58.8 s,
47.5 ran mostly on cpu0 and 23.9 mostly on cpu1.

Largest RSS at 152 s: `opera` 54 MB, `natp` 45 MB, `jciMMUI` 19 MB, `jciVBS` 12 MB, `jciSXMS`
10 MB, `bds` 10 MB.

**At the first SSH login.** Cumulative utime+stime from `/proc/<pid>/stat` at uptime 57–63 s, two
boots, HomeScreen at 32–33 s (GUI and SM tweaks). About 78 % of both cores was busy up to that point
(`/proc/uptime`: idle 26–30 s of 59–64 s × 2).

| Process | CPU s | Note |
|---|---|---|
| `ffxread01` | 12.1 | FlashFX read kernel thread, all system time |
| `opera` | 9.1–9.8 | HMI GUI |
| `natp` | 2.9–5.7 | |
| `L_jciBLMSetting` | 3.1–3.5 | |
| `syslog-ng` + `process-log` | 3.5 | system log path |
| `L_jciVBS`, `L_jciMMUI`, `vim_app`, `init` | 2.3–2.8 each | |

### Boot log

- One boot, `running_log` up to ~60 s: 11 585 lines (1.67 MB). Info 10 757 (93 %), Error 368,
  Warn 367, Note 43, Debug 29, Critical 12.
- Top zones: `VWM` 1335, `MMUI_SYSTEM` 680, `CPP_TIME` 580, `BLM_SETTINGS` 560, `MMUI_COMMON` 497,
  `BCA` 414, `VBS_BUS_VWM` 373, `BLM_CARPLAY` 340.
- Another pair of boots: about 8 300 lines (1.3 MB) in the first ~55 s, 91 % Info. After boot the
  rate drops to about 5 KB/s, and syslog-ng + process-log then use about 1.7 % of one core.
- `running_log` rotates at 512 KB, about every 1.5 min, into `log_NNNN.tar.gz`; ten are kept. Boot
  lines survive only for roughly the first 15 minutes of uptime.
- The live syslog-ng configuration is `/data/log/syslog-ng.conf` (writable `/data`). Destinations:
  `d_ram_log` (process-log, 512), `d_error_log`, and `d_console` = `/dev/console` (ttymxc1 UART,
  115200 baud). Every line also goes to the UART: ~7.7 MB by uptime 1199 s. The UART has no flow
  control, so excess lines are dropped rather than blocking.
- The GUI's own `Log.js` output is not a boot cost: only 10–43 lines per boot pass its level gate.
- Running-log timestamps are the time the line was written, in batches. A kernel line stamped
  41.45 s was written ~1 s later. Treat log times as ±1 s.
- Log level settings: [[Boot Speed Levers]], "Log level"; [[Syslog-ng]].

### Per-boot caches and tmpfs content

One boot, ~5 min uptime:

- `/tmp` (tmpfs, 93 MB) holds ~2.8 MB after boot. `/var`, `/root`, `/mnt`, `/media` and
  `/etc/resolv.conf` are rootfs symlinks into `/tmp`.
- Files rebuilt every boot, with creation time:

  | t (s) | File |
  |---|---|
  | 15 | `/tmp/devmgr_config_edit.xml` (VBS `sed` from a template) |
  | 16 | `/tmp/radio_database` (52 KB copy of `/data/radio/radio_database`) |
  | 27 | `/tmp/BdsConfiguration.xml` |
  | 48 | `/tmp/vui_emnavi_poi.eco` |
  | 54 | `/tmp/db1_con_cache.sqlite` (400 KB), `/tmp/db1_ch_cache.sqlite` |
  | 57 | `/tmp/vui_contacts0.eco`, `/tmp/sms.db`, `/tmp/email.db` |

  All files after ~45 s come later than HomeScreen and the Android Auto projection.
- The GStreamer registry is rebuilt every boot ([[Boot Speed Levers]], "GStreamer registry").
- RACE (USB media) keeps its resume DB only in tmpfs (`/mnt/tmp/race/database/RACE_ResumePlay.db`).
  At boot the restore finds nothing (`Stat failed`) and costs no time.
- Fail-safe version: `jciUpdatea` runs `jci-failsafe-version.sh` (31.3 → 34.0 s in the log). Its
  fast path reads 22 bytes at offset 7339776 of `/dev/mtd7`. On an idle system that `dd` takes
  ~0.03 s and the MTD name lookup ~0.07 s, so the 2.7 s is scheduling delay under boot load.
- Opera maps every font in `OPERA_FONTS` (`/jci/fonts/CHN`, 17 TTF files including CJK, Arabic,
  Hebrew, Thai) at start-up. Its disk cache is off (`[Disk Cache] Enabled=0`). Presto keeps no
  compiled-JS cache on disk.
- No `/etc/ld.so.cache` and no fontconfig on the rootfs.
- Album art and Gracenote start order: [[Media Album Art]].

## How to: measure a boot

- SM events: copy `/tmp/smevents.txt` after each boot.
- Screens: grep `ctxtChg` in `/var/log/running_log/log` within ~15 minutes of the boot, before the
  lines rotate out. Rotated lines are in the gzip tar archives in the same directory:
  `tar -xzOf log_0000.tar.gz | grep -a <text>`. GUI lines carry the tag
  `GUI_<APP>[pid] <Level> (<file>:<line> )`.
- Stopwatch against the log: screen dark to kernel start is about 5 s. Measured from the moment
  the screen went dark on a `GenericReboot(0)` (b46):

  | Event | Stopwatch (s) | Log (s) | Difference (s) |
  |---|---|---|---|
  | Home ready (`Recv ctxtChg:HomeScreen`) | 37.6 | 32.4 | 5.2 |
  | Boot animation visible / `tranlogo is started` | 34.1 | 25.2 | 8.9 |
  | Android Auto on screen / `AAP_PROJECTION_SETUP` | 63.8 | 54.4 | 9.4 |

  - The animation becomes visible ~3.7 s after `tranlogo is started`.
  - The Android Auto picture comes ~4 s after `AAP_PROJECTION_SETUP`.
  - In that boot, Home still responded slowly at 41.8 s on the stopwatch.
  - From a cold power-on (stopwatch from the power button), the Android Auto item turned active at
    41 s, against 34.4 s in the log: ~6.6 s for the button-to-log offset plus GUI render.

## Verified on

Firmware `cmu150_EU_74.00.324A`, legacy board, `sm.conf`. Single-boot numbers can differ by a few
seconds between boots; see "Variation between boots".

## Open questions

- What holds the Disclaimer `Timeout` for 9–11 s against its 3.5 s timer, beyond the GUI app
  initialisation.
- What resets the affinity of `dbus-daemon` and the `vim_app` main thread after a move to cpu1.
- Whether the wireless adapter's detach is its own timeout or a phone-link event.
