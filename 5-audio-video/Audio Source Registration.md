---
tags: [cmu150, audio, audio-manager, dbus, kb]
---

# KB — CMU150 Audio Source Registration

How an audio source (a "stream") is registered with, and selected through, the Audio Manager
(AM) on the Mazda CMU150, firmware `74.00.324A`. Companion to
[[Audio Architecture]] (stack overview, ALSA devices) and the `jci-ui` notes
([[04-app-anatomy]] for Entertainment menu rows, [[02-messaging-bus]] for MMUI).

> **Scope:** stock firmware only (Lua sources/bytecode, ELF strings, `sm.conf`, `asound.conf`).
> Lines marked *(inferred)* are derived from constants or symbol names. Paths are on-device
> rootfs paths.

---

## 1. Components

| Piece | Path | Form |
|---|---|---|
| Audio Manager daemon | `/usr/bin/audioMgr.lua` | Lua 5.1 **bytecode** (`release/7462_/src/audioMgr.lua`), started by SM |
| Source matrix (stream types + arbitration) | `/usr/share/audio_management/audioMgr_SourceMatrix_JCI.lua` | Lua bytecode |
| Control logic (per-stream mux/VBS actions) | `/usr/share/audio_management/audioMgr_ControlLogic_JCI.lua` | Lua bytecode |
| Audio Control daemon (mux/mixer over ALSA) | `/usr/bin/audioCtrl.lua` + `audioCtrl_Config_JCI.lua` | Lua bytecode |
| Amplifier daemon | `/usr/bin/audioAmp.lua` + `audioAmp_Config_JCI.lua` | Lua bytecode |
| Client library | `/jci/lib/libaudiomanagement.so` (`audiomanagement.c`) | C, exports `AUDIO_AMSERVICE_*` wrappers |
| Sample/test client | `/jci/am/am-client` | ELF; opens 3 sessions, registers 3 streams, exercises focus |
| Shared Lua helpers | `/usr/share/lua/5.1/xse/{audioUtils,dbusUtils,commonUtils,pluginUtils,coroutineLock}.lua` | `audioUtils.lua` bytecode (holds bus/object names) |

`/jci/am/SourceMatrix.lua`, `ControlLogic.lua`, `ControlConfig.json` are **plain-text, older-format
copies and are not loaded**: the SM launch line (§2) points at `/usr/share/audio_management/`.
The `/jci/am/` copies list fewer streams (`Pandora`, `Stitcher`, `BTMUSIC`, `USB`, `iPod`, `AM`,
`FM`, `Navi`, `BTHF`, `VR`) and a single `Cabin` destination.

## 2. Launch / configuration

SM (`/jci/sm/sm.conf`) starts, in dependency order: `audio_config`
(`/jci/am/audioconfigupdate.sh`, creates the `/data/asound.conf` symlink if absent) →
`audio_start` (`start_audio.sh`, runs `dbus-tracker` on both buses) → `audio_control` → 
`audio_amplifier` → `audio_manager`.

The `audio_manager` entry runs `/usr/bin/audioMgr.lua` with the matrix file
`-m /usr/share/audio_management/audioMgr_SourceMatrix_JCI.lua`, the control logic
`-l /usr/share/audio_management/audioMgr_ControlLogic_JCI.lua`, the buses
`-g`/`--app unix:path=/tmp/dbus_service_socket` and `--hmi unix:path=/tmp/dbus_hmi_socket`,
and the ready flag `/tmp/dbus/com.xsembedded.service.AudioManagement`.

- Matrix and control-logic files are fixed command-line arguments in the SM config; both live on
  the read-only rootfs. No `/data` or `/data_persist` override path for them was found.
- Both audio services carry `reset_board="yes"`: an exit of `audio_manager` (or
  `audio_control`/`audio_amplifier`) is configured to reboot the board.
- The AM is **not** a per-app registry. Sources are discovered at runtime: each client registers
  its own stream over D-Bus.

## 3. D-Bus surface

Two buses, both `dbus-daemon` instances with `<allow_anonymous/>` and an allow-all default policy
(`/etc/dbus-1/service.conf`, `hmi.d/jci_hmi.conf`, `service.d/jci_service.conf`); the service bus
also listens on `tcp:port=3000,bind=*`.

| Bus | Socket | Object on `com.xsembedded.service.AudioManagement` | Role |
|---|---|---|---|
| service (APP) | `/tmp/dbus_service_socket` | `/com/xse/service/AudioManagement/AudioApplication` | source clients: sessions, streams, focus |
| HMI | `/tmp/dbus_hmi_socket` | `/com/xse/service/AudioManagement/HmiMode` | HMI side: source (mode) selection |

Interface string used by the helpers: `com.xsembedded.ServiceProvider`. Related services:
`com.xse.service.AudioControl` (`/com/xse/service/AudioControl`, methods `setMux`/`getMux`/
`setMixer`/`getMixer`), `com.xsembedded.service.AudioAmplifier`
(`/com/xse/service/AudioAmplifier`), `com.xsembedded.service.xsesvc2dbus`
(`/com/xse/service/xsesvc2dbus`, VBS gateway: `SetMute`, `SetUnMute`, `SourceSelect`,
`GetActiveSource`, `GetMuteStatus`).

### 3.1 APP-side methods (source clients)

Payloads are JSON strings (`cjson`). Names and fields as found in the AM:

| Method | Payload fields | Notes |
|---|---|---|
| `openSession` | `busName`, `objectPath`, `destination` | Returns a numeric `sessionId`. `busName`/`objectPath` is where the AM calls the client back. Destination default is `Cabin`. |
| `registerAudioStream` | `sessionId`, `streamName`, `streamModeName`, `streamType`, `focusType` | `focusType` ∈ `permanent`, `transient`. `streamModeName` empty = not selectable by HMI ("modeable" prompt in AM debug menu). One stream per session. |
| `requestAudioFocus` | `sessionId`, `requestType`, `forceReq` | `requestType` ∈ `request`, `query` (would-it-be-granted), `queue`. Error `sessionId %d MUST be transient` exists for a non-transient session on this path. |
| `audioActive` | `sessionId`, `playing` | Client signals real audio start/stop; `audioActive(false)` has a timeout (`iAudioActiveFalseTimeout`) after which the AM calls it internally. |
| `abandonAudioFocus` | `sessionId`, `reason` | |
| `closeSession` | `sessionId` | |

Error names returned: `ERR_INV_OP`, `ERR_INV_SESSION`, `ERR_DUP_STREAMNAME` (stream names are
unique per destination — "Duplicates not allowed"), `ERR_INV_STREAMTYPE` (type not in the
destination's request matrix), `ERR_INV_FOCUSTYPE`, `ERR_AUDIO_DISABLED`, `ERR_INV_MODE`.

AM → client callbacks (on the `busName`/`objectPath` given at `openSession`):
`onAudioFocusChange` (`gained` / `lost` / `temporarilyLost`, also mixed variants) and
`onRequestAudioFocusResult` (`granted` / `failed` / `abandoned` / would-fail / would-be-granted).
Both carry `streamName`, `streamType`, `newFocus`; the AM suppresses a callback if the session is
already in that state.

### 3.2 HMI-side methods

`openSessionHMI`, `closeSessionHMI`, `getMode`, `getModeList`, `setMode` (payload
`{"sessionId":N,"newMode":"<streamModeName>"}`). Signals to the HMI session: `onModeChange`
(`activeMode`), `onModeListChange` (`modeList`). `setMode` makes the AM find the session whose
stream registered that `streamModeName` (`FindModeSession`, only modeable streams are searched)
and run a focus request for it; a failed switch returns
`ERROR: Unable to change to %s at this time`. `ChangeToPrevMode` exists for "switch back to the
previous source". The native MMUI system UIA (`libjciuiasystem.so`) links `libaudiomanagement.so`
and exports `openSessionHMI`/`setMode` usage *(inferred from symbols)*.

## 4. Stream types, arbitration

Types are defined in the **request matrix** of the (bytecode) source matrix; a `registerAudioStream`
with a type absent from the destination's matrix fails with `ERR_INV_STREAMTYPE`. Type names seen
in the loaded matrix: `Media`, `CPMedia`, `AAMedia` (`TypeGroup1`, mutually exclusive group),
`InfoUser`, `InfoMix`, `InfoSystem`, `LowAlert`, `VR`, `Telephony`, `HighAlert`, `MP911`,
`IncomingRg`, `CPAlerts`, `CPInCall`, `CPTel`, `CPAltAudio`, `CPSiri`, `AAGuidance`,
`AAGuidanceNoMix`, `AAVR`; priority-matrix names `SYSVR`, `BTRVR`, `EMAIL`, `PHONEBOOK`,
`BTHFVOL`, `RINGTONEVOL`, `Navi`, `NaviAlt`, `NAVIVOL`. Verbs: `Allow`, `Switch`, `Deny`,
`Suspend`, `Mix`, `NA`, plus priority verbs (`Prio_Switch`); semantics in
[[Audio Architecture]] §3. `Media` requested while `Media` active = `Switch`
(current stream terminated → client receives `lost`).

The matrix also supports `typeOverrides` and `nameOverrides` (per-stream-name action overrides)
and a DiPO/CarPlay check service (`DipoCheckRequest`/`DipoCheckResult` on its own bus name).

## 5. Control logic: from focus event to audio path

`ControlLogic` (bytecode) is keyed `Cabin → <streamName> → <event>` with events `pre_gained`,
`post_gained`, `pre_gainedMIX`, `pre_lost`, `pre_temporarilyLost`, `audioActive_true/false`,
`pre_granted`, `pre_timeout`. A stream name with **no** entry is accepted: the AM logs
`Stream %q (Destination %q) has no control logic. (not an error)` and performs no mux/VBS action.

The generic action (identical in the readable older copy and in the loaded bytecode) on
`pre_gained`: mute (`SetMute`) → 50 ms → `AudioControl.setMux{name,input}` → VBS `SourceSelect`
→ 50 ms; on `pre_lost`: mute → `setMux{input="off"}`; `audioActive_true` → unmute.

Stream names present in the loaded control logic: `LM AM FM HAR CD AUX TV DVD XM USB USBVIDEO iPod
Pandora Stitcher SCREENREP AHA BTMUSIC TA DAB MLENT BTECA BTECAALERT BTHF BTHFALERT SYSVR BTRVR
EMAIL SMS PHONEBOOK BTHFVOL RINGTONEVOL CPINCALL CPTELEPHONY CPALERTS CPSIRI CPMEDIA AAVR AAMedia
CPDEFAULT CPDEFAULTIN Navi NaviAlt NAVIVOL CPALTAUDIO AAGuidance AAGuidanceNoMix`.
Constants referenced for the entertainment path: mux `entertainment_mux` with inputs
`entertainmentMuxUsbIn`, `entertainmentMuxBtsaIn`, `entertainmentMuxMlIn`, VBS name `CMUMEDIA`.
Exact stream→input pairing in the loaded file is *(inferred)*: `USB`/`USBVIDEO`/`iPod` →
`UsbIn`; `Pandora`/`Stitcher`/`AHA`/`BTMUSIC` → `BtsaIn` (this pairing is verbatim in the older
text copy: Pandora, Stitcher, BTMUSIC → `BtsaIn`/`BT-A`; USB, iPod → `UsbIn`; AM, FM → own
inputs); `MLENT` → `MlIn`.

## 6. Mux = softvol ALSA PCMs

`/etc/asound.conf` (→ `/data/asound.conf`, templates `.LHD`/`.RHD`) comments: *"These ALSA cards and
mixer elements are part of the Entertainment Mux"*. Each mux input is a **named softvol PCM** whose
control element carries the mux-input name; `AudioControl.setMux` switches the elements:

| PCM (write here) | softvol control (mixer element, card 0) | Slave | Writer on stock firmware |
|---|---|---|---|
| `entertainmentUsb` | `entertainmentMuxUsbIn` | `generic_asymed` | `/jci/usbm/usbms` (USB media) |
| `entertainmentBtsa` | `entertainmentMuxBtsaIn` | `generic_asymed` | `/jci/bds/bds` (Bluetooth audio) |
| `entertainmentMl` | `entertainmentMuxMlIn` | `generic_asymed` | none found (comment: "for mirror link media streams") |
| `carplayMainAudio` | `carplayMuxCarPlayIn` | CarPlay chain | CarPlay |
| `androidautoMainAudio` / `…VR` | `aautoMuxAAutoIn` / `aautoMuxAAutoVRIn` | AA chain | Android Auto |
| `informationGeneric`, `informationNavi`, `informationNaviAlt` | `informationMux*In` | — | TTS / navigation prompts |

All `entertainment*` PCMs are `plug` front-ends: any input sample rate, 2 ch (`route_policy
duplicate`), `min_dB -100 / max_dB 0`. `audioCtrl` config lists `entertainment_mux` inputs
`UsbIn, BtsaIn, MlIn, AMIn, FMIn` (`AMIn`/`FMIn` handled by an `extHdlr` script, tuner path),
mixer `IHU_Mixer` with muxes `entertainment`, `information`, `vr`, `telephony`, `alert`,
`carplay`, `carplayinformation`, `aauto`, `aautoinformation`.
`entertainmentMl` is referenced by no JCI binary.

## 7. Stock sources: how they plug in

- Each source's native business-logic module opens its own AM session on the service bus and
  registers a stream: `/jci/radio/svcjciradio.so`, `btmusic/svcjcibtmusic.so`,
  `usbm/blmjciusbm.so`, `cd`, `dvd`, `auxin`, `dab`, `xm`, `xmaudio`, `bthf`, `bteca`, `btrvr`,
  `navi`, `pb`, `msg`, `carplay/blmjcicarplay.so`, `aapa/blmjciaapa.so` (each contains the string
  `registerAudioStream`). The audio payload itself goes to the ALSA PCM of §6 (e.g. `usbms`,
  `bds`).
- Pandora, Stitcher, Aha: UIA libraries `libjciuiapandora.so` (uiaid 10), `libjciuiastitcher.so`
  (29), `libjciuiaaharadio.so` (30), loaded by `/jci/mmui/mmui_config.xml`. Their stream names
  (`Pandora`, `Stitcher`, `AHA`) use the Bluetooth-audio mux input; there is no separate network
  audio engine for them in the AM.
- The GUI never talks to the AM. Entertainment rows carry `audioSourceId` + `mmuiEvent`
  (`SelectRadioFM`, `SelectUSBA`, `SelectPandora`, …); the MMUI system UIA
  (`/jci/mmui/uia/libjciuiasystem.so`) implements each as a compiled handler
  (`GUIIFM_Process_System_Select<X>_Evt` → `UIA_SYSTEM_Event_Select<X>`) and an
  `AudioSourceManager` (`selectApp`, `setLastModeStack`, reconnect timers, `GoToActiveSource`).
  The set of `Select*` events is fixed in that library; an unknown `mmuiEvent` string has no
  handler.
- The UI learns the active source from MMUI message `StatusUpdateAudioSource`
  (`payload.statusAudio.audioSource`, handled in `systemApp.js` → Now-Playing icon on the row whose
  `audioSourceId` matches); the value set is native.
- Hardware keys reach the Opera UI as DOM key events (see the input architecture note); track
  next/previous are consumed by whichever UIA app/context is current.

## 8. Requirements for a source client

- A client needs: a D-Bus connection to `/tmp/dbus_service_socket` that **owns a bus name and
  exports a method handler** (for the callbacks), then `openSession` → `registerAudioStream` →
  `requestAudioFocus` → (on `gained`) write PCM to an `entertainment*` PCM and call `audioActive` →
  `abandonAudioFocus`/`closeSession`. `am-client` and `libaudiomanagement.so` implement exactly
  this sequence.
- Registration needs no change to any AM config file as long as `streamType` exists in the matrix
  (`Media`) and `streamName` is unused. An unlisted name gets arbitration (`Media`×`Media` =
  `Switch`, so the previous source receives `lost`) but **no mux/VBS action** (§5).
- Using a stream name that already has control logic and whose stock client is not running
  (e.g. `MLENT`, `AHA`, `DAB`, `SCREENREP`, `HAR`, `DVD`, `AUX`) gives mux selection without
  editing control logic; those names must not already be registered by a stock module
  (`ERR_DUP_STREAMNAME`).
- Source selection from the HMI side is `HmiMode.setMode(streamModeName)` on the HMI bus; any
  HMI-bus client can open an HMI session (§3), the bus policy being allow-all.
- Adding a new name to the control logic / a new type to the matrix / a new native `Select*` event
  or UIA library requires changing rootfs files (`/usr/share/audio_management/*`,
  `/jci/mmui/mmui_config.xml`, `/jci/sm/sm.conf`, compiled UIA `.so`); no `/data` override was
  found for any of them. `/data/asound.conf` (symlink target, writable partition) is the only
  audio configuration file located on a writable path.

## 9. Open items

1. Exact D-Bus wire form: method names/arg signatures of the `ServiceProvider` interface
   (JSON string argument(s) per method), and the callback method names/signature at the client's
   object path — defined in `libaudiomanagement.so` / `dbusUtils.lua` generic-request handlers.
2. Which `streamModeName` values stock modules register, and whether the MMUI system UIA drives
   `setMode` for the Entertainment rows (names visible via `getModeList` at runtime).
3. Which stock stream names are registered at runtime on a unit with no phone/USB attached
   (determines which names are free: `ERR_DUP_STREAMNAME`).
4. Exact `streamName → mux input / VBS name` table in the loaded (bytecode) control logic;
   behaviour of `setMux` + `SourceSelect` for `MLENT`/`AHA` with no stock writer; whether an
   unknown VBS source name is accepted by the vehicle-bus gateway.
5. Whether PCM written to `entertainmentMl` reaches `hw:0,0` audibly when `entertainmentMuxMlIn`
   is selected (softvol default state, `generic_asymed` chain, DSP routing).
6. `focusType` that media streams use (`permanent` vs `transient`), and the `audioActive` timeout
   value (`iAudioActiveFalseTimeout`) in the loaded matrix; `ERR_AUDIO_DISABLED` conditions
   (`bDisableAudioAtStartup`).
7. Whether the MMUI system UIA reflects a non-stock stream in Now-Playing / status bar / last-source
   resume (`lastModeStack`) at all, or only for its compiled source list; what happens to the
   current UIA app context when AM focus moves to an unknown stream.
8. Behaviour of hardware next/previous/volume keys when the AM-active stream is not one of the
   UIA-owned sources (volume path is ALSA/amplifier-side; track keys are UIA-context-side).
9. D-Bus access from a non-SM process: socket ownership/mode (the service-bus policy file
   `service.d/jci_service.conf` has `allow user="*"`, `allow own="*"` and eavesdrop enabled; the
   HMI-bus file was not compared), and the effect of a client that disappears while holding focus
   (AM cleanup on bus-name loss).
10. Consequence of an AM exit given `reset_board="yes"` (e.g. malformed request crashing the
    Lua daemon).
