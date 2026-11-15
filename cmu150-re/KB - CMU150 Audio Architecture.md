---
tags: [cmu150, audio, alsa, kb]
---

# KB — CMU150 Audio Architecture

How audio works on the Mazda CMU150 (firmware `74.00.324A`): the stack from a source down to
the amplifier, how the stock players drive it, and the interface for feeding PCM. Source
registration details are in [[KB - CMU150 Audio Source Registration]]; display in
[[KB - CMU150 Display Architecture]].

> **Scope:** the stock audio stack of firmware `74.00.324A`. Lines marked *(inferred)* are
> derived from footprint and known behaviour, not stated verbatim in the firmware. Paths are
> on-device rootfs paths.

---

## 1. The stack (top → bottom)

```
app UI (Opera/JS)  ──control/metadata──►  native service  ──►  Audio Manager (arbitration)
                                                                     │  grants a stream
                                                                     ▼
          GStreamer 0.10 pipeline  ──PCM──►  ALSA (dmix_48000)  ──►  hw:0,0 DAC  ──►  DSP  ──►  amplifier
```

- **The UI never touches audio bytes.** A JS app only sends transport events and receives
  metadata (see §5). All real audio is native.
- **The Audio Manager (AM) owns the speakers.** Nothing reaches the amp as a first-class
  source without the AM granting a *stream* and arbitrating it (§3).
- **The actual sink is ALSA** (`dmix_48000` → `hw:0,0`), fed by **GStreamer 0.10** (§2, §4).

---

## 2. Audio stack components

**ALSA** — `libasound.so.2`, tools `aplay` / `arecord` / `amixer` / `speaker-test`,
config `/usr/share/alsa/*`, and `/etc/asound.conf` → symlink to **`/data/asound.conf`**
(writable copy; templates `/etc/asound.conf.LHD` and `.RHD` for left/right-hand drive).

**GStreamer 0.10** — full set under the lib dirs: `libgstreamer-0.10`, `libgstapp-0.10`
(**appsrc/appsink** → you can push/pull buffers), `libgstaudio`, `libgstbase`, `libgstcdda`,
`libgstfsl-0.10` (**Freescale i.MX** hardware codecs), plus `gst-launch-0.10`. This is the
media-pipeline framework the native players use.

**No PulseAudio** — `/etc/pulse` empty, no `libpulse`. The `/jci/pa/svcjcipa.so` "**PA**" is
**JCI's own audio service**, not PulseAudio. Don't assume a PulseAudio API.

**JCI native audio services / libs:**
- `/jci/am/` — **Audio Manager**: `SourceMatrix.lua` (sources + interrupt matrix),
  `ControlConfig.json`, `ControlLogic.lua`, `AmplifierConfig.lua`, `am-client`,
  `start_audio.sh`.
- `/jci/audioplayer/libjciaudioplayer.so` — the native file/media player (GStreamer-based).
- `/jci/pa/svcjcipa.so` — JCI primary-audio service + `testapp_pa`.
- `/jci/dsp/` — per-vehicle/amplifier DSP tuning XMLs (`dsp_J*-Bose.xml`, `-NoBose.xml`, …).
- `/jci/lib/libaudiomanagement.so`, `libaudiomatrixservice.so`, `libaudiorecordingservice.so`,
  `libaudiocoder*.so` (wav/tts coders).

---

## 3. Audio Manager: stream types + arbitration

`/jci/am/SourceMatrix.lua` does **not** list apps — it defines semantic **stream types** and
a matrix of who may interrupt whom. Verbatim stream types:

`Media · Tone · InfoSystem · InfoMix · InfoUsr · VR · MedAlr · Telephony · HighAlr · MP911`

Matrix verbs (per *requested × current* pair): `Allow` (play, suspend current), `Switch`
(play, terminate current), `Deny`, `Suspend`, `Mix`, `NA` (default = deny). Examples from the
`Cabin` destination matrix:
- `Media` requested vs active `Media` → **Switch** (new music replaces old).
- `Telephony` vs `Media` → **Suspend** Media (call ducks/holds music).
- `MP911` → overrides essentially everything.

Music enters as a **`Media`** stream; the AM arbitrates it against nav/voice/calls/alerts and
routes it through DSP → amp. Ducking, mixing and DSP routing apply only to audio that is a
`Media` stream.

---

## 4. ALSA devices — the concrete PCM interface

From `/etc/asound.conf` (→ `/data/asound.conf`; template `.LHD`). Hardware DAC = **`hw:0,0`**,
**48000 Hz, S16_LE**. Named PCMs:

| PCM device | Type | Role |
|---|---|---|
| **`dmix_48000`** | dmix (slave `hw:0,0`, buffer 6144 / period 1536) | **main music output**; software-mixes *multiple* writers |
| `dmix_16000` | dmix | 16 kHz (voice) |
| `dmix_48000_cptel` | dmix | CarPlay telephony |
| `dmix_48000_siri` | dmix | Siri |
| `dmix_48000_loopWrite` | dmix (slave `hw:Loopback2,0,0`) | write into the **ALSA Loopback** card |
| `dsnoop_48000` (+ `_cptel`, `_siri`) | dsnoop | capture side of `hw:0,0` |
| `dsnoop_48000_loopRead` | dsnoop (slave `hw:Loopback2,1,0`) | read back the loopback |
| `asymed` | asym | pair: playback `dmix_48000` + capture `dsnoop_48000` |
| `fileout` | file (slave `dmix_48000`) | debug: writes WAV to `/tmp/debug.wav` |

**Key facts:**
- `dmix` allows **concurrent writers**, so an app can open `dmix_48000` and write S16_LE/48k
  stereo and get sound immediately — `aplay -D dmix_48000 file.wav` proves the path.
- There is an **ALSA Loopback** (`hw:Loopback2`): audio written to `*_loopWrite` can be read
  from `*_loopRead`. *(inferred)* this is how the AM / CarPlay / telephony capture and
  re-route external streams — likely the "clean" external-source entry point.
- `rate_converter = speexrate`; a `fileout` sink exists for debugging.

---

## 5. UI ↔ native messaging (how players are controlled)

The Opera UI talks to native services over **WebSockets** (`/jci/gui/common/js/GuiConfig.js`):

| Socket | URI | Protocol |
|---|---|---|
| mmui | `ws://127.0.0.1:2700` | `ihu-guiifm-protocol` |
| appsdk | `ws://127.0.0.1:2800` | `ihu-appsdk-protocol` |
| dbapi | `ws://127.0.0.1:2766` | `ihu-dbapi-protocol` |

- Send: `framework.sendEventToMmui(...)` (e.g. `SelectUSBA`, play/pause). Receive via
  `framework.routeMmuiMsg` / `routeAppsdkMsg` → app message handlers.
- Media handlers an app gets back: `PlayerStatus`, `ElapsedTime`, `ShowStationInfo`,
  `ShowInfo`, `NowPlaying*` (observed in `stitcherApp.js`).
- **AppSDK is plugin-based** (`/jci/appsdk/appsdk_plugin_config.xml` + `svcjciappsdk.so`);
  plugins present: `usbm` (USB media), `cd`, `sxms` (SiriusXM), `fav`, `msg`, `navifav`.
  **There is no generic "play arbitrary audio/PCM" plugin** — media plugins only control
  *existing* sources.

---

## 6. How the stock USB player works (end-to-end)

1. **USB insert** → native **USB manager** (`/jci/usbm`, `/jci/usbmgr`) mounts the stick and
   indexes media into a metadata DB (playlists/tracks; `USBM_MetadataType_*`).
2. **UI** `apps/usbaudio/js/usbaudioApp.js` ↔ **usbm** via AppSDK plugin
   `libjcipluginusbm.so` (appsdk `:2800`): browse playlists/songs, send transport
   (play/pause/skip), receive `PlayerStatus`/`ElapsedTime`/NowPlaying.
3. **Source select** → `mmuiEvent SelectUSBA/USBB` → **AM** grants a **Media** stream.
4. **Decode + output (native):** `libjciaudioplayer.so` builds a **GStreamer 0.10** pipeline
   — representative form *(inferred)* `filesrc ! decodebin ! audioconvert ! audioresample !
   alsasink` — writing PCM to **`dmix_48000` → `hw:0,0`** → DSP → amplifier.

Chain: **GStreamer → ALSA `dmix_48000` → `hw:0,0` → DSP → amp**, under AM arbitration.

### "Connected"/streaming apps (Aha / Stitcher / Pandora)
Each is a **thin JS UI only** (`<name>App.js` + css + a logo). The audio comes from a
**companion app on the paired phone** over **Bluetooth A2DP / USB** *(inferred, matches known
MZD behaviour + the tiny on-device footprint)*; the head unit shows metadata and sends
transport. The AM treats it as a `Media` source (BTAudio/USB). The unit does **not** stream
these from the internet itself.

---

## 7. Feeding audio from a native player

Two native approaches exist (a JS app alone cannot emit sound):

**A. GStreamer 0.10 (matches the stock player):**
```
souphttpsrc location=<url> ! decodebin ! audioconvert ! audioresample ! alsasink device=dmix_48000
```
(`appsrc` instead of `souphttpsrc` if you push your own buffers.)

**B. Decode in the player** and write S16_LE/48k PCM to `dmix_48000` via `libasound`.

**Politeness tradeoff:**
- **Direct `dmix_48000`** → sound immediately, even bypassing the AM. No ducking for
  nav/calls, wrong or no DSP routing, and it competes with the active source.
- **Through the AM** → request a `Media` stream via `am-client` / the matrix so arbitration,
  ducking and DSP apply. *(inferred)* the **Loopback** card is the likely external-source
  entry the AM expects.

---

## 8. Open questions

- Registration of a `Media` source is documented in [[KB - CMU150 Audio Source Registration]]
  (D-Bus sessions/streams/focus on the AM; the mux inputs are softvol PCMs
  `entertainmentUsb` / `entertainmentBtsa` / `entertainmentMl`, §6 there). That note supersedes
  §3 ("matrix lists no apps") and §7 ("Loopback as external entry") here.
- Whether writing `dmix_48000` directly actually reaches the amp (amp mute/enable may be
  gated by the AM) and how it behaves vs an active stock source.
- Role of the `Loopback2` card as the external-source entry; what the AM reads from it.
- Which codecs the on-board GStreamer 0.10 + Freescale plugins actually decode (MP3/AAC/FLAC/
  Opus?) for streaming use.
- Whether the writable `/data/asound.conf` is an intended configuration surface.
