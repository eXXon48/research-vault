---
tags: [cmu150, audio, alsa]
---

# Audio Architecture

The audio stack from a source down to the amplifier: components, ALSA devices, how the stock
players are controlled and how they output sound.

## Overview

```
app UI (Opera/JS) ──control, metadata──> native service ──> Audio Manager (arbitration)
                                                                 │ grants a stream
                                                                 v
        GStreamer 0.10 pipeline ──PCM──> ALSA PCM ──> hw:0,0 DAC ──> DSP ──> amplifier
```

- **The UI never handles audio data.** A JS app only sends transport events and receives metadata.
  All audio is native.
- **The Audio Manager (AM) owns the speakers.** A source plays as a first-class source only after
  the AM grants it a stream and arbitrates it against other sources. Registration and arbitration:
  [[Audio Source Registration]].
- **The sink is ALSA**, fed by GStreamer 0.10. Entertainment sources write to the softvol mux PCMs
  (`entertainmentUsb`, `entertainmentBtsa`, …); `dmix_48000` → `hw:0,0` is the shared output.
- There is no PulseAudio.

Related: [[Audio Source Registration]], [[Display Architecture]], [[Media Album Art]],
[[02-messaging-bus]].

## Reference

### Components

| Component | Path | Note |
|---|---|---|
| ALSA | `libasound.so.2`; tools `aplay`, `arecord`, `amixer`, `speaker-test`; config `/usr/share/alsa/*` | `/etc/asound.conf` is a link to `/data/asound.conf` (writable copy); templates `/etc/asound.conf.LHD` and `.RHD` for left- and right-hand drive |
| GStreamer 0.10 | `libgstreamer-0.10`, `libgstapp-0.10` (`appsrc`/`appsink`), `libgstaudio`, `libgstbase`, `libgstcdda`, `libgstfsl-0.10` (Freescale i.MX hardware codecs), `gst-launch-0.10` | media pipelines for the native players |
| Audio Manager | `/usr/bin/audioMgr.lua` with `/usr/share/audio_management/*` | Lua bytecode, started by SM; details in [[Audio Source Registration]] |
| Native player | `/jci/audioplayer/libjciaudioplayer.so` | GStreamer-based file and media player |
| Primary-audio service | `/jci/pa/svcjcipa.so`, `testapp_pa` | JCI's own service; "PA" is not PulseAudio |
| DSP tuning | `/jci/dsp/` | per-vehicle and per-amplifier XMLs (`dsp_J*-Bose.xml`, `-NoBose.xml`, …) |
| Libraries | `/jci/lib/libaudiomanagement.so`, `libaudiomatrixservice.so`, `libaudiorecordingservice.so`, `libaudiocoder*.so` (WAV and TTS coders) | — |

- `/etc/pulse` is empty and there is no `libpulse`.
- `/jci/am/` holds `SourceMatrix.lua`, `ControlLogic.lua`, `ControlConfig.json`,
  `AmplifierConfig.lua`, `am-client` and `start_audio.sh`. The `.lua` and `.json` files there are
  older plain-text copies; the AM does not load them ([[Audio Source Registration]],
  "Components").

### ALSA devices

From `/etc/asound.conf` (→ `/data/asound.conf`; template `.LHD`). The hardware DAC is `hw:0,0`,
48000 Hz, S16_LE.

| PCM | Type | Role |
|---|---|---|
| `dmix_48000` | dmix (slave `hw:0,0`, buffer 6144, period 1536) | main output; mixes several writers in software |
| `dmix_16000` | dmix | 16 kHz (voice) |
| `dmix_48000_cptel` | dmix | CarPlay telephony |
| `dmix_48000_siri` | dmix | Siri |
| `dmix_48000_loopWrite` | dmix (slave `hw:Loopback2,0,0`) | write into the ALSA Loopback card |
| `dsnoop_48000` (+ `_cptel`, `_siri`) | dsnoop | capture side of `hw:0,0` |
| `dsnoop_48000_loopRead` | dsnoop (slave `hw:Loopback2,1,0`) | read back from the loopback |
| `asymed` | asym | playback `dmix_48000` + capture `dsnoop_48000` |
| `fileout` | file (slave `dmix_48000`) | debug: writes WAV to `/tmp/debug.wav` |
| `entertainment*`, `carplay*`, `androidauto*`, `information*` | softvol | the AM mux inputs ([[Audio Source Registration]], "Mux PCMs") |

- `dmix` allows concurrent writers. `aplay -D dmix_48000 file.wav` plays immediately.
- `hw:Loopback2` is an ALSA Loopback card: audio written to `*_loopWrite` can be read from
  `*_loopRead`.
- `rate_converter = speexrate`.

### UI ↔ native messaging

The Opera UI talks to native services over WebSockets (`/jci/gui/common/js/GuiConfig.js`):

| Socket | URI | Protocol |
|---|---|---|
| MMUI | `ws://127.0.0.1:2700` | `ihu-guiifm-protocol` |
| AppSDK | `ws://127.0.0.1:2800` | `ihu-appsdk-protocol` |
| DBAPI | `ws://127.0.0.1:2766` | `ihu-dbapi-protocol` |

- Send with `framework.sendEventToMmui(...)` (for example `SelectUSBA`, play, pause). Receive
  through `framework.routeMmuiMsg` / `routeAppsdkMsg` → app message handlers.
- Media messages an app receives: `PlayerStatus`, `ElapsedTime`, `ShowStationInfo`, `ShowInfo`,
  `NowPlaying*` (seen in `stitcherApp.js`).
- AppSDK is plugin-based (`/jci/appsdk/appsdk_plugin_config.xml` + `svcjciappsdk.so`). Plugins:
  `usbm` (USB media), `cd`, `sxms` (SiriusXM), `fav`, `msg`, `navifav`. No plugin plays arbitrary
  audio; the media plugins only control existing sources.

### Stock USB player

1. **USB insert:** the native USB manager (`/jci/usbm`, `/jci/usbmgr`) mounts the stick and indexes
   the media into a metadata database (playlists, tracks; `USBM_MetadataType_*`).
2. **UI:** `apps/usbaudio/js/usbaudioApp.js` talks to `usbm` through the AppSDK plugin
   `libjcipluginusbm.so` (port 2800): browse playlists and songs, send transport commands, receive
   `PlayerStatus`, `ElapsedTime`, `NowPlaying`.
3. **Source select:** `mmuiEvent SelectUSBA` / `SelectUSBB`; the `usbm` business logic holds a
   `Media` stream in the AM.
4. **Output:** `/jci/usbm/usbms` writes PCM to the `entertainmentUsb` mux PCM → `hw:0,0` → DSP →
   amplifier.

### Streaming apps (Aha, Stitcher, Pandora)

Each is a thin JS UI only (`<name>App.js`, CSS and a logo). Their AM stream names (`Pandora`,
`Stitcher`, `AHA`) use the Bluetooth-audio mux input, and the AM has no network audio engine for
them. The unit does not stream these services from the internet itself.

## How to: feed audio from a native player

A JS app alone cannot output sound. Two native approaches:

**GStreamer 0.10**, the same framework the stock player uses:

```
souphttpsrc location=<url> ! decodebin ! audioconvert ! audioresample ! alsasink device=dmix_48000
```

Use `appsrc` instead of `souphttpsrc` to push buffers from the program.

**libasound**: decode in the player and write S16_LE / 48 kHz PCM with `libasound`.

| Output | Effect |
|---|---|
| `dmix_48000` directly | plays immediately, outside the AM: no ducking for navigation or calls, no DSP routing by the AM, mixes with the active source |
| an `entertainment*` mux PCM, with an AM stream | arbitration, ducking and mux selection apply ([[Audio Source Registration]], "Requirements for a source client") |

## Verified on

Stock rootfs of firmware `cmu150_EU_74.00.324A` (static analysis).

## Open questions

- Whether PCM written to `dmix_48000` directly always reaches the amplifier (amplifier mute and
  enable may be controlled by the AM), and how it interacts with an active stock source.
- What uses the `Loopback2` card, and whether the AM reads from it.
- Which codecs GStreamer 0.10 with the Freescale plugins decodes (MP3, AAC, FLAC, Opus).
- Whether the writable `/data/asound.conf` is meant as a configuration surface.
- The exact GStreamer pipeline that `libjciaudioplayer.so` builds.
