---
tags: [cmu150, boot, sm, services, ipc, reference]
---

# SM Service Graph

Every service the System Manager (`/jci/sm/sm`) runs: how `sm.conf` describes it, how SM handles
failures, what the services talk to, and which chains gate the Home screen, Android Auto (AA) and
CarPlay.

## Overview

- SM starts services in dependency order. `<dependency>` is the only ordering mechanism. The
  `stage_*` entries are empty markers that exist only to be depended on.
- Most services carry `reset_board="yes"`: a failure reboots the unit instead of degrading.
- Services also depend on each other at runtime through D-Bus client libraries, loopback TCP,
  unix sockets and POSIX mqueues. SM does not see these dependencies.
- The longest shared step for AA and Home is `jciMMUI`. AA also waits on the Bluetooth stack,
  only because `mfid` depends on `jciBCA`.

Related: [[Boot Timeline]] (timings), [[SM Service Control (smctl)]] (command-line tool),
[[Boot Speed Levers]], [[README|CMU150 research]].

Sources:

- the stock `sm.conf`;
- an ELF scan of every service binary and its `/jci/lib` dependencies (`DT_NEEDED`, dynamic
  symbols, strings);
- decompiles of `libdevicemanager.so`, `libjcibcacomm.so`, `libjciusbmgrcomm.so`,
  `libwifiplugin.so`, `blmjciaapa.so`, `svcjcimmui.so`, `svcjcirm.so` and `svcjciupdatea.so`;
- a running unit, read-only: D-Bus name owners on both buses, `/proc/<pid>/fd`, `/proc/net/tcp`,
  `/proc/net/unix`, `/dev/mqueue`;
- SM event dumps and `running_log` of one stock boot and one boot with modified dependencies.

## Reference

### `sm.conf` format

**Defaults and server settings.**

| Element | Values |
|---|---|
| `<services>` defaults | `type="process"`, `reset_board="no"`, `retry_count="0"`, `autorun="yes"`, `affinity="0x01"`, `ping_timeout="8000"`, `start_timeout="45000"`, `stop_timeout="5002"` (ms) |
| `<sm_server>` | `watchdog_enable="true"`, `reports_timeout="60000"`, `sigkill_wait_before_reboot="30"` |
| `<connect>` | `start_timeout="5003"`, `client_port_number="9091"` |
| other | SM pings every `10000` ms; `system_save_timeout="55000"` |

**Service types.** The legacy file has 117 services: 67 `jci_service` and 50 `process`.

- A `jci_service` runs as `sm_svclauncher -l <name> <lib.so> 0 -a <args>`. The process name is
  `L_<name>`, truncated to 15 characters.
- The `.so` exports `ServiceInit` / `ServiceRun` and calls `SM_SVC_SetUserData`,
  `SM_SVC_HandlePing` and `SM_SVC_IsRunning`.

**Service attributes.**

| Attribute | Meaning |
|---|---|
| `<dependency type="service" value="X"/>` | start only after X is `STARTED`; the only ordering mechanism |
| `<connection type="service">` | lists peers; does not order the start |
| `ready_flag` | the service counts as `STARTED` when this path exists: a `/tmp/dbus/<name>` file written by the D-Bus tracker (`xsesvc2dbus_service`, `audio_*`), or a POSIX mqueue (`dmserver` → `/dev/mqueue/ipc13398-0`, `dsp` → `ipc12415-0`) |
| `pid_file` / `external` | SM adopts a process started outside SM (`lvds_service`, `dbus_service`, `dbus_hmi`, `nativeguictrl`, `vim_app`) |
| `exit_allowed` / `started_after_exit_allowed` | the process may exit after start without counting as a crash (scripts; `mdnsd`, which forks and exits) |
| `autorun="no"` | started only on request; for example MMUI starts `tranlogo` |

Stages (`stage_1`, `stage_3`, `stage_usb`, `stage_navi`, `stage_vr`, `stage_wifi`, …) are short
`process` entries.

### SM event log

`/tmp/smevents.txt` is a CSV: `pid,name,event,seconds-since-kernel-start`. Every service inherits
its file descriptor.

| Event | Meaning |
|---|---|
| `SM_SVC_CMD_START` | SM starts the service |
| `SM_SVC_HANDSHAKE_DONE` | the launcher connected back to SM |
| `SM_SVC_CMD_STARTING` | — |
| `SM_SVC_CMD_STARTED` | the service is up |
| `SM_SVC_CMD_STOPPED` | the service stopped |

### Failure handling

- A service that misses pings (`ping_timeout`) or exits without `exit_allowed`:
  - while retries are left (`retry_count`), SM restarts it;
  - when they are used up, `reset_board="yes"` reboots the board (`sigkill_wait_before_reboot`
    seconds after SIGKILL), and `reset_board="no"` leaves the service stopped.
- A service that exceeds `start_timeout` is treated as failed, on the same path.
- Most `jci_service` entries carry `reset_board="yes"`. A service that fails to start in a changed
  order reboots the unit.
- Dependents of a stopped service never start; SM does not retry the dependency edge. A late or
  missing dependency silently drops a feature. Example: album art is lost when `gracenote_server`
  is late ([[Media Album Art]]).
- Opera ↔ MMUI websocket: the GUI reconnects every 500 ms. A close after the connection is up calls
  `_restartCMU` ([[01-architecture-and-boot]]).
- `devicemanager` `ServiceRun` is a `sleep(5)` loop → `vDevMgrCheckHealthStatus` →
  `SM_SVC_HandlePing`. The CD-mechanism code retries 5 times with sleeps of 1, 2, 5 and 10 s.

### D-Bus

Two `dbus-daemon` instances:

| Socket | Config | Names |
|---|---|---|
| `/tmp/dbus_service_socket` | `service.conf` | BLM / service names |
| `/tmp/dbus_hmi_socket` | `hmi.conf` | HMI-facing names |

Client libraries map one-to-one to bus names:

| Library | Bus name |
|---|---|
| `libjciaapa_client` | `com.jci.aapa` |
| `libjcibca_client` | `com.jci.bca` |
| `libjcibdsdbusclient` | `com.jci.bds` |
| `libjciamclient` | `com.xsembedded.service.AudioManagement` |
| `libjcisystem_client` | `com.jci.blm.system` |
| `libjcirm_consumer` | `com.jci.rm_ipc_cons` |
| `libjciupdatea_client` | `com.jci.updatea_sys_ipc` |
| `libjciusbmgr_client` | `com.jci.usbmgr` |
| `libjcisettings_client` | `com.jci.settings` |
| `libjcivbs*client` | `com.jci.vbs.*` |

A service that links a client library depends on the owner of that name at runtime, whether or
not `sm.conf` lists the dependency.

### Loopback TCP

| Port | Owner | Clients / role |
|---|---|---|
| 2700 | `jciMMUI` | Opera GUI (`ihu-guiifm-protocol` websocket) |
| 2800 | `jciAppsdk` | Opera (`ihu-appsdk-protocol`) |
| 2766 | `jciDbapi` | Opera (`ihu-dbapi-protocol`) |
| 5000 | `settings` | most BLM services (`--proxy=tcpip://`) |
| 9091, 2904 | SM | every service (launcher control) |
| 3722, 7000 | `carplayd` | CarPlay |
| 5354 | `mdnsd` | Bonjour (`libwifiplugin`) |
| 6010, 2011 | `natp` | `jciTTSPlayer`, `jciVUI` |
| 7035 | `vim_app` | input |

The websocket ports are defined in `/jci/gui/common/js/GuiConfig.js`.

### Unix sockets and mqueues

| Path | Owner | Role |
|---|---|---|
| `/tmp/MFISocket`, `/tmp/allgorpccomm<pid>` | `mfid` | Apple MFi authentication |
| `/tmp/RPCClient318` | `carplayd` | — |
| `/data/JVMM_*.sfd` | `JVMM` | navigation |
| `/data/JVMM_RS485.sfd` | `jciTds` | — |
| `/tmp/wpa_ctrl_<pid>-N` | `NMS_svc` | `wpa_supplicant` control |
| `/tmp/wayland-0` | compositor | display |
| `/dev/log` | syslog-ng | logging |

`libjcipc` uses POSIX mqueues `/ipcN-0` (requests) and `/ipcN-reply`. A queue that is also a
`ready_flag` makes the creating service the server.

### `devicemanager` comm libraries

`devicemanager` loads its comm libraries with `dlopen`. The paths come from
`/etc/devmgr_config_master.xml`, which VBS copies to `/tmp/devmgr_config_edit.xml` at ~15 s. The
compiled-in defaults (`/usr/lib/libusbmgrcomm.so`, `libbtmgrcomm.so`, `libwifimgr.so`) do not
exist; the XML overrides them.

| Key | Library | Init |
|---|---|---|
| `acBTLibPath` | `/jci/lib/libjcibcacomm.so` | `BTDevRegister`: a second `SM_Connect` + `SM_SubscribeForEvents`, an HMI-bus connection, signal enables only (`PA_PairingStatusResp`, `BCA_ConnectionStatusResp`, `BCA_CarPlayOOBPairingStatus`, …). No synchronous call to `com.jci.bca`, so `jciBCA` does not need to be up. |
| `acUSBLibPath` | `/jci/lib/libjciusbmgrcomm.so` | `initDBUS`: `JCIDBUS_conn_connect`, signal registration (`USBDevAttached` / `Detached` / `Event`), one synchronous `USBMGR_NotifyEnableBridge_sync` to `com.jci.usbmgr`, which needs `jciUSBMGR` (started at ~14 s) |
| `acWifiLibPath` | `/usr/lib/libwifiplugin.so` | `DNSServiceCreateConnection` to `mdnsd`, and a `select()` loop on its own thread |
| `boEnableAndroidAutoFlag` | `TRUE` | — |

`ServiceInit` runs these steps in order:

1. log and OSAL init, semaphores, config parse, six `race_msg_queue_init`;
2. `system("mkdir -p …")` and `system("chmod 775 …")`;
3. `eDeviceDbInit` (`/data/devmgr/DeviceDatabase.db`) and `eReadEAPProtocolList`;
4. `eWifiMgrInit`, `eBTMgrInit`, `eUsbMgrInit`;
5. worker threads.

### Bluetooth and CarPlay chain inputs

From the stock image:

- `stage_3.sh` and `stage_bt.sh` are empty scripts (`#!/bin/sh` only). `stage_3` depends on
  `jciMMUI`, so every service behind `stage_3` or `stage_bt` waits for the Home screen service.
- `stage_bt` depends on `stage_3` and `emmc_mount`. About 15 Bluetooth services wait for it:
  `dsp`, `jciAudioPlayer`, `jciBTHF`, `jciBTMUSIC`, `jciDTMF*`, `jciMSG*`, `jciPB`, `pbs`,
  `jcibteca`, `jciBCA`, `jciPA`.
- `dmserver` links only `libjcicommon`, `libjcipc`, `libjcixmlreader`, `libtrfs` and `libexpat`. It
  reads `/jci/devmgr/devmgr_config.xml` and keeps its store in `/data/devmgr` (`/data` is already
  used by `start_normal_mode.sh` before SM starts). Its stock dependency on `jciRadio` has no
  matching library or file dependency. Dependents: `bds`, `jciBCA`, `jcibteca`, `jcifav`.
- `bds` needs `dmserver` (reads its paired-device entries at start) and `settings`. Its start is
  about 2 s of load, about 3.5 s until the device entries are read, and about 3 s of UART/HCI and
  database init.
- `svcjcipa.so` (`jciPA`) links the client libraries of `jciAAPA`, `jciBCA`, `bds`, `jciCARPLAY`,
  `devicemanager`, `settings`, `system`, `vbs` and `blm_nms`.
- `blmjcicarplay.so` (`jciCARPLAY`) links, among others, the client libraries of `jcilvds_blm`,
  `jciCDRP`, `dsp`, `gracenote_server`, `jciRM`, `jciUSBMGR`, `audio_manager` and `mfid`
  (`libmfi`). `dsp` waits for `stage_bt`, so `jciCARPLAY` reaches `dsp` only because `jciPA` also
  waits for `stage_bt`.

### CPU affinity

- 2 cores at a fixed 996 MHz ([[Hardware and Firmware]], "CPU").
- `sm.conf` sets `affinity_mask="0x01"` on the `<services>` group, so every service is pinned to
  cpu0 unless it overrides it.
- 18 services override with `affinity_mask="0x02"` (cpu1): `jciBLMSettings`, `devices`,
  `jciNativegui`, `jciBUCPSA`, `jciVideoctrl`, `tranlogo`, `quicktranlogo`, `exitlogo`, `Opera`,
  `dsp_config`, `audio_config`, `audio_start`, `aap_service`, `carplayd`, `NNG`, `JVMM`, `jciTds`,
  `start_wifi`. The SM process itself runs on cpu1.
- On a running unit: 108 tasks pinned to cpu0, 19 to cpu1, 58 free (kernel threads, shells).

## Measurements

Times are seconds since kernel start (`STARTED` unless noted).

### Stock chain to Android Auto

`stage_1` → `settings` → `jciVBS` → `xsesvc2dbus_service` → `audio_control` → `audio_manager` →
`jciRadio`/`jciCD`/`jciauxin` → `jciMMUI` (20.4 → 24.5) → `stage_3` → `bds` (24.5 → 32.3) →
`jciPA` → `jciBCA` (33.5 → 34.6) → `mfid` → `devicemanager` (34.7 → 38.0) → `aap_service` →
`jciAAPA` (38.0 → 41.5). AA ready at 41.9.

The Bluetooth stack (`bds`, 7.7 s) sits on the AA path only because `mfid` depends on `jciBCA`.

### Chains with modified dependencies

Three dependencies changed:

- `mfid` → `stage_3`;
- `jciMMUI` → `audio_manager`, `audio_control` and `settings` (instead of `jciRadio`, `jciauxin`
  and `jciCD`);
- `jcinavi` and `JVMM` → `jciCARPLAY`.

| Chain | Path | Ready |
|---|---|---|
| Android Auto | `stage_1` 9.2 → `settings` 10.8 → `jciVBS` 15.2 → `xsesvc2dbus_service` 15.3 → `audio_control` 17.6 → `audio_manager` 18.2 → `jciMMUI` 23.2 → `stage_3` 23.2 → `mfid` 23.3 → `devicemanager` 29.9 → `aap_service` 29.9 → `jciAAPA` 32.7 | `[Androidauto]Available Status true` 33.1 |
| Next AA gates | `stage_3` → `emmc_mount` 24.3 → `jciUpdateS` 26.8 → `jciUpdatea` 28.7; `jciRM` 25.4 | 1.2 s slack behind `devicemanager` |
| Home screen | shared with AA up to `jciMMUI`; Opera 14.0, GUI ready 20.8 → `tranlogo` + IntroAnimation 23.9 → Disclaimer 28.6 (0.6–1.9 s after `tranlogo` stops) → DisposeIntroVideo 29.4 (GUI app-load queue drained) | `Recv ctxtChg:HomeScreen` 29.9 (stock 37.4) |
| CarPlay | `stage_3` → `bds` 32.3 → `jciPA` 34.4 → `jciCARPLAY` 40.1 | `carplay_feature_available value=2` 40.4 (stock 45.9) |

- `jciMMUI` (18.2 → 23.2, 5 s) is the longest step shared by AA and Home.
- `devicemanager` (6.5 s) is the longest step on AA alone.
- `bds` (9.1 s) heads the CarPlay chain.

### `devicemanager` start

During its 4.8 s from `STARTING` to `STARTED`, `devicemanager`:

- stays at 6 threads (the workers are not created yet);
- uses ~0.16 cpu-s;
- is seen in D state;
- reads 0.75 MB in total over the boot.

It waits on `ffx01` NAND reads ([[FlashFX Read Path]]), not on a peer timeout or on CPU. Other
services starting in the same window add to that read queue.

### Services inside the AA, Home and CarPlay window that do not gate them

| Service | Start → started | Overlaps | Effect of a later start |
|---|---|---|---|
| `jciTestDiag`, `jciusbdtc` | 15.2 → 17.0 | `jciVBS` → `audio_control` | diagnostics / DTC later |
| `jciDAB`, `jciCD`, `jciauxin` | 18.2 → 19.3 | `jciMMUI` | source screens later; MMUI modules use SM started-callbacks; the GUI loads the last source app before Home |
| `jcifav`, `jciaudiosettings`, `jcilvds_blm` | 23.3 → 26.6 | `devicemanager` (NAND-bound) | favourites / audio settings later |
| `start_network` → `start_wifi`, `NMS_svc` | 24.5 → 31.4 | `devicemanager`, `jciAAPA` | Wi-Fi AP later: wireless AA / CarPlay delayed |
| `dsp`, `jciAudioPlayer`, `jciDTMF*` | 24.5 → 26.7 | `devicemanager` | Bluetooth calls (HFP) later |
| `gracenote_server` | 24.6 | `devicemanager` | album art lost (no retry in `jciBTMUSIC` / `usbms`) |
| `jciCDRP` | 29.9 → 32.7 | `jciAAPA` | USB music database later |
| `natp` → `jciTTSPlayer`, `jciVUI` → `kapture` | 45 → 68 | none (after AA and CarPlay) | none on these chains; `natp` reads ~40 MB |

Most of these carry `reset_board="yes"`. A service moved behind a later dependency must still
start inside `start_timeout`, and must not break a client-library bus dependency, which SM does not
see.

### CPU during boot

`/proc/stat`, 9.2–31.2 s after power-on:

| Core | Idle | Other |
|---|---|---|
| cpu0 | 0.1 s of 22 s | user 8.6, sys 11.0, softirq 1.6, iowait 0.6 |
| cpu1 | 4.2 s | iowait 4.0 |

cpu0 is the CPU bottleneck during boot, while cpu1 has spare time.

## Verified on

Firmware `cmu150_EU_74.00.324A`, legacy board (`/jci/sm/sm.conf`).
