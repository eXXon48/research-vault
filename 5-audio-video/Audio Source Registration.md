---
tags: [cmu150, audio, audio-manager, dbus]
---

# Audio Source Registration

How an audio source (a "stream") is registered with the Audio Manager (AM), how it is arbitrated
and selected, and how the AM switches the audio path.

## Overview

- The AM is a Lua daemon. Its stream types, arbitration matrix and control logic are Lua bytecode
  files on the read-only rootfs.
- The AM is not an app registry. Each source client opens a session over D-Bus and registers its own
  stream at run time.
- When a stream gains focus, the control logic for its name mutes, switches the entertainment mux
  (softvol ALSA PCMs) and tells the vehicle bus the new source.
- A new client needs no AM config change if it uses an existing stream type (`Media`) and an unused
  stream name. It then gets arbitration but no mux switching, unless it uses a stream name that
  already has control logic.

Related: [[Audio Architecture]] (stack, ALSA devices), [[04-app-anatomy]] (Entertainment menu rows),
[[02-messaging-bus]] (MMUI), [[SM Service Graph]].

Sources: Lua sources and bytecode, ELF strings, `sm.conf`, `asound.conf` of the stock firmware.
Lines marked *(inferred)* are derived from constants or symbol names.

## Reference

### Components

| Piece | Path | Form |
|---|---|---|
| Audio Manager daemon | `/usr/bin/audioMgr.lua` | Lua 5.1 bytecode (`release/7462_/src/audioMgr.lua`), started by SM |
| Source matrix (stream types, arbitration) | `/usr/share/audio_management/audioMgr_SourceMatrix_JCI.lua` | Lua bytecode |
| Control logic (per-stream mux and VBS actions) | `/usr/share/audio_management/audioMgr_ControlLogic_JCI.lua` | Lua bytecode |
| Audio Control daemon (mux and mixer over ALSA) | `/usr/bin/audioCtrl.lua` + `audioCtrl_Config_JCI.lua` | Lua bytecode |
| Amplifier daemon | `/usr/bin/audioAmp.lua` + `audioAmp_Config_JCI.lua` | Lua bytecode |
| Client library | `/jci/lib/libaudiomanagement.so` (`audiomanagement.c`) | C, exports `AUDIO_AMSERVICE_*` wrappers |
| Sample client | `/jci/am/am-client` | ELF; opens 3 sessions, registers 3 streams, exercises focus |
| Shared Lua helpers | `/usr/share/lua/5.1/xse/{audioUtils,dbusUtils,commonUtils,pluginUtils,coroutineLock}.lua` | `audioUtils.lua` is bytecode (holds bus and object names) |

`/jci/am/SourceMatrix.lua`, `ControlLogic.lua` and `ControlConfig.json` are plain-text copies in an
older format and are not loaded: the SM launch line points at `/usr/share/audio_management/`. The
`/jci/am/` copies list fewer streams (`Pandora`, `Stitcher`, `BTMUSIC`, `USB`, `iPod`, `AM`, `FM`,
`Navi`, `BTHF`, `VR`) and a single `Cabin` destination.

### Launch and configuration

SM (`/jci/sm/sm.conf`) starts, in dependency order:

1. `audio_config` (`/jci/am/audioconfigupdate.sh`): creates the `/data/asound.conf` link if absent;
2. `audio_start` (`start_audio.sh`): runs `dbus-tracker` on both buses;
3. `audio_control`;
4. `audio_amplifier`;
5. `audio_manager`.

The `audio_manager` entry runs `/usr/bin/audioMgr.lua` with:

| Argument | Value |
|---|---|
| `-m` (matrix) | `/usr/share/audio_management/audioMgr_SourceMatrix_JCI.lua` |
| `-l` (control logic) | `/usr/share/audio_management/audioMgr_ControlLogic_JCI.lua` |
| `-g` / `--app` | `unix:path=/tmp/dbus_service_socket` |
| `--hmi` | `unix:path=/tmp/dbus_hmi_socket` |
| ready flag | `/tmp/dbus/com.xsembedded.service.AudioManagement` |

- The matrix and control-logic paths are fixed arguments in the SM configuration, and both files are
  on the read-only rootfs. No `/data` or `/data_persist` override exists for them.
- The audio services carry `reset_board="yes"`: an exit of `audio_manager`, `audio_control` or
  `audio_amplifier` reboots the board.

### D-Bus surface

Two buses, both `dbus-daemon` instances with `<allow_anonymous/>` and an allow-all default policy
(`/etc/dbus-1/service.conf`, `hmi.d/jci_hmi.conf`, `service.d/jci_service.conf`). The service bus also
listens on `tcp:port=3000,bind=*`.

| Bus | Socket | Object on `com.xsembedded.service.AudioManagement` | Role |
|---|---|---|---|
| service (APP) | `/tmp/dbus_service_socket` | `/com/xse/service/AudioManagement/AudioApplication` | source clients: sessions, streams, focus |
| HMI | `/tmp/dbus_hmi_socket` | `/com/xse/service/AudioManagement/HmiMode` | HMI side: source (mode) selection |

The helpers use the interface string `com.xsembedded.ServiceProvider`. Related services:

| Name | Object | Methods |
|---|---|---|
| `com.xse.service.AudioControl` | `/com/xse/service/AudioControl` | `setMux`, `getMux`, `setMixer`, `getMixer` |
| `com.xsembedded.service.AudioAmplifier` | `/com/xse/service/AudioAmplifier` | — |
| `com.xsembedded.service.xsesvc2dbus` | `/com/xse/service/xsesvc2dbus` | VBS gateway: `SetMute`, `SetUnMute`, `SourceSelect`, `GetActiveSource`, `GetMuteStatus` |

#### Source-client methods (service bus)

Payloads are JSON strings (`cjson`).

| Method | Payload fields | Notes |
|---|---|---|
| `openSession` | `busName`, `objectPath`, `destination` | returns a numeric `sessionId`. `busName`/`objectPath` is where the AM calls the client back. Destination default is `Cabin`. |
| `registerAudioStream` | `sessionId`, `streamName`, `streamModeName`, `streamType`, `focusType` | `focusType` ∈ `permanent`, `transient`. An empty `streamModeName` = not selectable from the HMI. One stream per session. |
| `requestAudioFocus` | `sessionId`, `requestType`, `forceReq` | `requestType` ∈ `request`, `query` (would it be granted), `queue`. The error `sessionId %d MUST be transient` exists on this path for a non-transient session. |
| `audioActive` | `sessionId`, `playing` | the client signals real audio start and stop. `audioActive(false)` has a timeout (`iAudioActiveFalseTimeout`), after which the AM calls it internally. |
| `abandonAudioFocus` | `sessionId`, `reason` | — |
| `closeSession` | `sessionId` | — |

| Error | Meaning |
|---|---|
| `ERR_INV_OP` | invalid operation |
| `ERR_INV_SESSION` | unknown session |
| `ERR_DUP_STREAMNAME` | stream names are unique per destination ("Duplicates not allowed") |
| `ERR_INV_STREAMTYPE` | type not in the destination's request matrix |
| `ERR_INV_FOCUSTYPE` | invalid `focusType` |
| `ERR_AUDIO_DISABLED` | audio disabled |
| `ERR_INV_MODE` | invalid mode |

Callbacks from the AM, on the `busName`/`objectPath` given at `openSession`:

| Callback | Values |
|---|---|
| `onAudioFocusChange` | `gained`, `lost`, `temporarilyLost`, and mixed variants |
| `onRequestAudioFocusResult` | `granted`, `failed`, `abandoned`, would-fail, would-be-granted |

Both carry `streamName`, `streamType` and `newFocus`. The AM suppresses a callback if the session is
already in that state.

#### HMI methods (HMI bus)

| Method or signal | Payload |
|---|---|
| `openSessionHMI`, `closeSessionHMI` | — |
| `getMode`, `getModeList` | — |
| `setMode` | `{"sessionId":N,"newMode":"<streamModeName>"}` |
| `ChangeToPrevMode` | switch back to the previous source |
| signal `onModeChange` | `activeMode` |
| signal `onModeListChange` | `modeList` |

- `setMode` makes the AM find the session whose stream registered that `streamModeName`
  (`FindModeSession`; only selectable streams are searched) and run a focus request for it.
- A failed switch returns `ERROR: Unable to change to %s at this time`.
- The MMUI system module (`libjciuiasystem.so`) links `libaudiomanagement.so` and uses
  `openSessionHMI` / `setMode` *(inferred from symbols)*.

### Stream types and arbitration

Stream types are defined in the request matrix of the source matrix (bytecode). A
`registerAudioStream` with a type that is not in the destination's matrix fails with
`ERR_INV_STREAMTYPE`.

- **Request-matrix types:** `Media`, `CPMedia`, `AAMedia` (`TypeGroup1`, mutually exclusive),
  `InfoUser`, `InfoMix`, `InfoSystem`, `LowAlert`, `VR`, `Telephony`, `HighAlert`, `MP911`,
  `IncomingRg`, `CPAlerts`, `CPInCall`, `CPTel`, `CPAltAudio`, `CPSiri`, `AAGuidance`,
  `AAGuidanceNoMix`, `AAVR`.
- **Priority-matrix names:** `SYSVR`, `BTRVR`, `EMAIL`, `PHONEBOOK`, `BTHFVOL`, `RINGTONEVOL`,
  `Navi`, `NaviAlt`, `NAVIVOL`.

Verbs, for each pair of requested × current stream:

| Verb | Effect |
|---|---|
| `Allow` | play; suspend the current stream |
| `Switch` | play; terminate the current stream (its client receives `lost`) |
| `Deny` | refuse |
| `Suspend` | suspend |
| `Mix` | play mixed with the current stream |
| `NA` | not applicable (default: deny) |
| `Prio_Switch` | priority switch |

Examples: `Media` requested while `Media` is active → `Switch` (new music replaces the old);
`Telephony` vs `Media` → `Media` suspended; `MP911` overrides almost everything.

The matrix also supports `typeOverrides` and `nameOverrides` (per-stream-name action overrides),
and a CarPlay check service (`DipoCheckRequest` / `DipoCheckResult` on its own bus name).

### Control logic

The control logic (bytecode) is keyed `Cabin → <streamName> → <event>`. Events: `pre_gained`,
`post_gained`, `pre_gainedMIX`, `pre_lost`, `pre_temporarilyLost`, `audioActive_true`,
`audioActive_false`, `pre_granted`, `pre_timeout`.

A stream name with no entry is accepted. The AM logs
`Stream %q (Destination %q) has no control logic. (not an error)` and performs no mux or VBS action.

The generic action, the same in the older text copy and in the loaded bytecode:

| Event | Action |
|---|---|
| `pre_gained` | mute (`SetMute`) → 50 ms → `AudioControl.setMux{name,input}` → VBS `SourceSelect` → 50 ms |
| `pre_lost` | mute → `setMux{input="off"}` |
| `audioActive_true` | unmute |

Stream names in the loaded control logic: `LM AM FM HAR CD AUX TV DVD XM USB USBVIDEO iPod Pandora
Stitcher SCREENREP AHA BTMUSIC TA DAB MLENT BTECA BTECAALERT BTHF BTHFALERT SYSVR BTRVR EMAIL SMS
PHONEBOOK BTHFVOL RINGTONEVOL CPINCALL CPTELEPHONY CPALERTS CPSIRI CPMEDIA AAVR AAMedia CPDEFAULT
CPDEFAULTIN Navi NaviAlt NAVIVOL CPALTAUDIO AAGuidance AAGuidanceNoMix`.

For the entertainment path the constants are: mux `entertainment_mux` with inputs
`entertainmentMuxUsbIn`, `entertainmentMuxBtsaIn`, `entertainmentMuxMlIn`; VBS name `CMUMEDIA`.

| Stream names | Mux input |
|---|---|
| `USB`, `USBVIDEO`, `iPod` | `UsbIn` |
| `Pandora`, `Stitcher`, `AHA`, `BTMUSIC` | `BtsaIn` |
| `MLENT` | `MlIn` |
| `AM`, `FM` | own inputs |

The pairing is verbatim in the older text copy for Pandora, Stitcher, BTMUSIC (`BtsaIn` / `BT-A`),
USB, iPod (`UsbIn`), AM and FM. For the loaded file it is *(inferred)*.

### Mux PCMs

`/etc/asound.conf` (→ `/data/asound.conf`, templates `.LHD` / `.RHD`) says: *"These ALSA cards and
mixer elements are part of the Entertainment Mux"*. Each mux input is a named softvol PCM whose
control element carries the mux-input name. `AudioControl.setMux` switches the elements.

| PCM (write here) | softvol control (card 0) | Slave | Stock writer |
|---|---|---|---|
| `entertainmentUsb` | `entertainmentMuxUsbIn` | `generic_asymed` | `/jci/usbm/usbms` (USB media) |
| `entertainmentBtsa` | `entertainmentMuxBtsaIn` | `generic_asymed` | `/jci/bds/bds` (Bluetooth audio) |
| `entertainmentMl` | `entertainmentMuxMlIn` | `generic_asymed` | none (comment: "for mirror link media streams"); no JCI binary references it |
| `carplayMainAudio` | `carplayMuxCarPlayIn` | CarPlay chain | CarPlay |
| `androidautoMainAudio` / `…VR` | `aautoMuxAAutoIn` / `aautoMuxAAutoVRIn` | Android Auto chain | Android Auto |
| `informationGeneric`, `informationNavi`, `informationNaviAlt` | `informationMux*In` | — | TTS, navigation prompts |

- All `entertainment*` PCMs are `plug` front-ends: any input sample rate, 2 channels
  (`route_policy duplicate`), `min_dB -100`, `max_dB 0`.
- The `audioCtrl` configuration lists `entertainment_mux` inputs `UsbIn`, `BtsaIn`, `MlIn`, `AMIn`,
  `FMIn`. `AMIn` and `FMIn` are handled by an `extHdlr` script (tuner path).
- Mixer `IHU_Mixer` has the muxes `entertainment`, `information`, `vr`, `telephony`, `alert`,
  `carplay`, `carplayinformation`, `aauto`, `aautoinformation`.

### Stock sources

- Each source's native business-logic module opens its own AM session on the service bus and
  registers a stream. Modules that contain the string `registerAudioStream`:
  `/jci/radio/svcjciradio.so`, `btmusic/svcjcibtmusic.so`, `usbm/blmjciusbm.so`, `cd`, `dvd`,
  `auxin`, `dab`, `xm`, `xmaudio`, `bthf`, `bteca`, `btrvr`, `navi`, `pb`, `msg`,
  `carplay/blmjcicarplay.so`, `aapa/blmjciaapa.so`. The audio data goes to a mux PCM (for example
  from `usbms` or `bds`).
- Pandora, Stitcher and Aha are MMUI modules `libjciuiapandora.so` (uiaid 10),
  `libjciuiastitcher.so` (29) and `libjciuiaaharadio.so` (30), loaded through
  `/jci/mmui/mmui_config.xml`. Their stream names use the Bluetooth-audio mux input.
- The GUI never talks to the AM. Entertainment rows carry `audioSourceId` + `mmuiEvent`
  (`SelectRadioFM`, `SelectUSBA`, `SelectPandora`, …). The MMUI system module
  (`/jci/mmui/uia/libjciuiasystem.so`) implements each as a compiled handler
  (`GUIIFM_Process_System_Select<X>_Evt` → `UIA_SYSTEM_Event_Select<X>`) and runs an
  `AudioSourceManager` (`selectApp`, `setLastModeStack`, reconnect timers, `GoToActiveSource`). The
  set of `Select*` events is fixed in that library; an unknown `mmuiEvent` string has no handler.
- The UI learns the active source from the MMUI message `StatusUpdateAudioSource`
  (`payload.statusAudio.audioSource`). `systemApp.js` shows the Now Playing icon on the row whose
  `audioSourceId` matches. The set of values is native.
- Hardware keys reach the Opera UI as DOM key events. Track next and previous are handled by the
  current MMUI module and context.

### Requirements for a source client

1. Connect to `/tmp/dbus_service_socket`, own a bus name and export a method handler for the
   callbacks.
2. `openSession` → `registerAudioStream` → `requestAudioFocus`.
3. On `gained`: write PCM to an `entertainment*` PCM and call `audioActive`.
4. When done: `abandonAudioFocus`, `closeSession`.

`am-client` and `libaudiomanagement.so` implement this sequence.

- Registration needs no AM config change as long as `streamType` exists in the matrix (`Media`) and
  `streamName` is unused. An unlisted name gets arbitration (`Media` × `Media` = `Switch`, so the
  previous source receives `lost`) but no mux or VBS action.
- A stream name that already has control logic, and whose stock client is not running (for example
  `MLENT`, `AHA`, `DAB`, `SCREENREP`, `HAR`, `DVD`, `AUX`), gets mux selection without editing the
  control logic. A stock module must not already have registered that name
  (`ERR_DUP_STREAMNAME`).
- From the HMI side, the source is selected with `HmiMode.setMode(streamModeName)` on the HMI bus.
  Any HMI-bus client can open an HMI session; the bus policy is allow-all.
- A new name in the control logic, a new type in the matrix, a new native `Select*` event or a new
  MMUI module requires changing rootfs files: `/usr/share/audio_management/*`,
  `/jci/mmui/mmui_config.xml`, `/jci/sm/sm.conf`, compiled module `.so` files. None of them has a
  `/data` override. `/data/asound.conf` is the only audio configuration file on a writable path.

## Verified on

Stock rootfs of firmware `cmu150_EU_74.00.324A` (static analysis).

## Open questions

1. The exact D-Bus wire form: method names and argument signatures of the `ServiceProvider`
   interface (JSON string arguments per method), and the callback method names and signature at the
   client's object path. They are defined in the generic-request handlers of
   `libaudiomanagement.so` / `dbusUtils.lua`.
2. Which `streamModeName` values the stock modules register, and whether the MMUI system module
   calls `setMode` for the Entertainment rows (visible with `getModeList` at run time).
3. Which stock stream names are registered at run time with no phone or USB stick attached; this
   decides which names are free.
4. The exact stream name → mux input / VBS name table in the loaded control logic. The behaviour of
   `setMux` + `SourceSelect` for `MLENT` / `AHA` with no stock writer. Whether the vehicle-bus
   gateway accepts an unknown VBS source name.
5. Whether PCM written to `entertainmentMl` is audible when `entertainmentMuxMlIn` is selected
   (softvol default state, `generic_asymed` chain, DSP routing).
6. The `focusType` that media streams use (`permanent` or `transient`), the `audioActive` timeout
   (`iAudioActiveFalseTimeout`) in the loaded matrix, and the conditions for `ERR_AUDIO_DISABLED`
   (`bDisableAudioAtStartup`).
7. Whether the MMUI system module shows a non-stock stream in Now Playing, the status bar or
   last-source resume (`lastModeStack`), and what happens to the current module context when AM
   focus moves to an unknown stream.
8. How hardware next, previous and volume keys behave when the active AM stream is not one of the
   module-owned sources. Volume is on the ALSA / amplifier side; track keys are on the module side.
9. D-Bus access from a process outside SM: socket owner and mode (`service.d/jci_service.conf` has
   `allow user="*"`, `allow own="*"` and eavesdropping enabled; the HMI-bus file was not compared),
   and AM cleanup when a client holding focus disappears from the bus.
10. The effect of an AM exit, given `reset_board="yes"` (for example a malformed request crashing the
    Lua daemon).
