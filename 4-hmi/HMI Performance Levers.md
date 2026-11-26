# HMI performance levers

What sets the speed of the stock HMI (firmware 74.00.324A, EU flavor), read from the stock
rootfs. Boot-time levers are in the System Manager config; run-time levers are in the Opera
GUI framework. Timings on a live unit are not part of this note.

## CPU

- Kernel 3.0.35 SMP, 2 cores activated (`Machine: JCI i.MX 6Quad Core CMU Board`, SoC
  identified as i.MX6Q rev 1.2).
- Kernel command line presets `lpj=8495104` → 1699.02 BogoMIPS per core (HZ=100).
- The i.MX CPU frequency driver is built in (`i.MXC CPU frequency driver` in dmesg) and is the
  thermal cooling device (`anatop_thermal_probe: default cooling device is cpufreq!`).
  `cpufreq_userspace.ko` and `cpufreq_stats.ko` exist as modules; no stock script loads them and
  no stock script writes `scaling_governor`.
- Live read (bench unit, 74.00.324A): both cores run governor `userspace` (the only governor
  available) at 996 MHz = `cpuinfo_max_freq`; available steps 996000 / 792000 / 396000 kHz.
  There is no frequency headroom to gain.
- Per-core load on a running unit (8 min after boot, idle on the home screen): core 0 ~35 % busy,
  core 1 ~5 % over a 5 s sample; since boot core 0 ~37 %, core 1 ~11 %. Services without an
  `affinity_mask` run on core 0 (the `<services>` default `0x01`).
- The running System Manager is `/jci/sm/sm -f /jci/sm/sm.conf -e /tmp/smevents.txt`
  (`sm_WCP.conf` is not used on this unit). Largest RSS: `opera` ~58 MB, `natp` ~43 MB,
  `L_jciMMUI` ~19 MB, `L_jciVBS` ~12 MB.
- I/O scheduler: `noop` (default).

## System Manager: core affinity

`/jci/sm/sm.conf` and `/jci/sm/sm_WCP.conf` set a default `affinity_mask="0x01"` (core 0) on the
`<services>` element. Services with `affinity_mask="0x02"` (core 1) in stock:

`jciBLMSettings`, `devices`, `jciNativegui`, `jciBUCPSA`, `jciVideoctrl`, `tranlogo`,
`quicktranlogo`, `exitlogo`, `Opera`, `dsp_config`, `audio_config`, `audio_start`,
`aap_service`, `carplayd`, `NNG`, `JVMM`, `jciTds`, `start_wifi`.

Opera (`/jci/opera/opera -u /jci/gui/index.html`) is pinned to core 1 only, together with
`JVMM`, `jciTds`, `carplayd`, `aap_service` and the audio setup services.

## System Manager: services started in the EU config

`autorun="no"` in stock: `tranlogo`, `quicktranlogo`, `exitlogo`, `NNG`, `jcixm`,
`jciXMDATA`, `jcixmad`. Everything else autostarts, including services for features that are
region- or hardware-specific: `irsstitcher_server`, `blm_ira_pandora`, `blm_ira_stitcher`,
`blm_ira_aha`, `jcidvd`, `jcitv`, `jciSXMS`, `jciXMMGR`.

Stage gates (`stage_wifi`, `stage_vr`, `stage_gap2`) depend on `blm_ira_aha`; `stage_wifi`
gates `start_wifi`, `NMS_svc` and `blm_nms_svc`.

`sm_WCP.conf` differs from `sm.conf` only in `args` (`wcp`, `new_hw`, `-wcp`, `GPS=UBLOX`)
and two extra dependencies (`stage_bt` on `emmc_mount`, `jciLDS` on `stage_navi`).

`/jci/scripts/jci-wifiap.sh` has `sleep 2` at lines 62 and 76 (start path, after `udhcpd` and
`hostapd`) and at 92 and 105 (stop path).

## Opera preferences

`/jci/opera/opera_home/opera.ini` (rootfs, read-only):

| Section | Key | Stock |
|---|---|---|
| `[Disk Cache]` | `Enabled` / `Size` | `0` / `2000` |
| `[Cache]` | `Figure` / `Document` | `8000` / `1000` |
| `[User Prefs]` | `Video RAM Limit` | `98304` |
| `[User Prefs]` | `Enable WebGL` | `1` |
| `[User Prefs]` | `Console Error Log Enabled` | `0` |
| `[Multimedia]` | `Show Animation` | `1` |

## GUI framework: screen transitions

Screen changes run CSS animations. `/jci/gui/common/transitions/js/TransitionsGen.js` maps
each named transition to a function and a `duration_ms`:

| Transition | `duration_ms` |
|---|---|
| `Crossfade`, `Back Depart + …`, `Explode + …`, `SB Slide + Crossfade …` | 600 |
| `UMP Depart + Crossfade + UMP Arrive` (and `+ SB Slide`) | 500 |
| `FastCrossfade`, `Slide Left/Right *`, other `UMP Depart + …` | 400 |
| `Fadeover` | 200 |
| `CrossfadeTo/FromCarplay`, keyboard variants | 100 |
| `Snap` | 0 |

`Crossfade` is also the fallback when no transition resolves.

The animations are CSS classes in `/jci/gui/common/transitions/css/TransitionsGen.css` named
`Transitions_<effect>_<delay>_<duration>` (e.g. `.Transitions_fadeIn_300_300` =
`animation-duration: 300ms; animation-delay: 300ms`). 44 classes; durations 50–600 ms, delays
0–400 ms. The CSS uses unprefixed `animation-*` properties.

Completion is event-driven: `Transitions.js` listens for `animationend` (lines ~872, ~1029).
`duration_ms` only sets a watchdog timer (`duration_ms + _WATCHDOG_TIMER_MARGIN_MS`, line ~211)
and the timer-based cleanup used by UMP slides (`_setCleanupByTimer`, called from
`TransitionsGen.js` with fixed values 0, 400 and 500 ms; it is a plain prototype method, not
bound). `_WATCHDOG_TIMER_MARGIN_MS` is 500. All 44 animation classes set
`animation-fill-mode: both`. The status bar slide
takes its delay/duration as JS literals (`statusBar.transitionVisible(0, 300, …)`).

Opera 12.10 (Presto 2.12.388, desktop build) accepts `CSSStyleRule.style.setProperty(
"animation-duration", "150ms", "")` at run time and reads the value back normalized to
seconds (`0.15s`), so transition timings can be rescaled in place without a new stylesheet.

## GUI framework: logging cost

`/jci/gui/framework/js/Log.js`, `Log.prototype._log`:

1. Returns early only if the level is below `logDefaults.__global__` (stock `"info"`,
   `/jci/gui/common/js/logDefaults.js`).
2. Otherwise builds `new Error().stack` and runs a regex over it to find the caller's file and
   line, builds a prefix with `new Date().toISOString()`, joins the arguments and calls the
   native `JCILogger.JCILog()` plugin (`guiConfig.pcLogging = false` on target).

So every `log.info`/`log.warn`/`log.error` call on target costs a stack capture + regex + a
native log write; `log.debug` returns at step 1. Call sites in `/jci/gui`: 2117 `log.info`,
618 `log.warn`, 218 `log.error`, 3345 `log.debug`. `Websockets.js` logs on many inbound MMUI
messages. The per-app levels in `logDefaults` are only checked when `guiConfig.pcLogging` is
true. `guiConfig.disableLogs` only calls `Debug.disableDragonfly()` and does not gate `Log`.

The gate is a run-time object: `log._logLevels["__global__"]` (numeric: debug 1, info 2,
warn 3, error 4, fatal 5) can be raised after `Log` is constructed.

## GUI framework: Websockets.js stack helpers

`/jci/gui/framework/js/Websockets.js` declares `_isStack` (line 813) and `_delStack` (line 840)
as global functions, but calls them as methods (`this._isStack(…)`, `this._delStack(…)`,
lines 260–297) in the MMUI `onmessage` handler. They are not on `Websockets.prototype`, so each
call throws `TypeError`, which aborts the rest of that `onmessage` run. The calls are reached
when a context-change, focus-stack or transition message arrives while `_waitMsgArray` is
not empty (an out-of-order sequence counter).

Defining both on the prototype (same bodies) removes the exception. The bodies compare
`_waitMsgArray[i][0]` with the literals `"focusStack"` / `"transition"`, while the array holds
message strings, so they return `false` in practice; the effect of the fix is that the rest
of the handler runs.

### The wait queue loses messages (stock 74.00.324A, md5 `426032e998ca371914360217fff01993`)

Read from the stock file; not yet confirmed in a unit log. With or without the prototype fix:

- **Queued `focusStack` / `transition` are never delivered on their set.** They are pushed as
  `[jsonStr, seq]` (lines 310, 323, 340); `ctxtChg` is pushed as a bare string (line 286).
  `_isStack` compares element `[0]` (the whole JSON text) with the literal `"focusStack"` /
  `"transition"`, so it never matches and `_delStack` never removes anything.
- **The force flush drops them.** Every 50 received messages (`mRecvMsgCounterLimit`, line 30)
  the handler passes each queued element to `_doRouteMmuiMsg` (line 216). An array element is
  coerced to `"<json>,<seq>"`, `JSON.parse` fails, and the message is discarded with
  `Unable to parse incoming MMUI message`. Queued `ctxtChg` strings parse and are delivered
  late. The counter is reset to 0 when the first element enters an empty queue, so the flush
  comes 50 messages after that.
- **A `focusStack` with no `ctxtChg` in its set is always queued.** The expected seq
  (`mMmuiWebsocketWaitSeqCounter`) changes only when a `ctxtChg` arrives on an empty queue or
  when a message is queued. A focus-only change (new seq, no screen change) differs from it,
  so it is queued and later dropped by the flush.
- **A non-empty queue holds back screen changes.** While the queue is not empty, a `ctxtChg`
  whose seq differs from the expected seq is queued without updating it (lines 279-286), and
  the rest of that set follows it into the queue. Screen changes then wait for the flush.
- **A `transition` before its `focusStack`** (`mDoFocus != seq`, line 320) is queued and is lost
  the same way. A lost `transition` with `enabled:false` leaves the framework in the transition
  state (see jci-ui/02-messaging-bus).
- **After a GUI restart** the seq globals (lines 27-28) start again at 0 while the native side
  keeps its counter, so the first `focusStack` / `transition` sets mismatch and are queued.

Not established from the GUI side: whether the native sender emits each set's messages in seq
order on one socket, which would make arrival-order delivery (no queue) correct.

A replacement handler that strips the sequence byte with the stock rule (`_isTargetMsg` on
the raw frame, drop the last character) and routes every frame at once was checked in desktop
Opera 12.10 (Presto): the routed text equals the stock extraction for ctxtChg, focusStack,
transition and non-target frames (seq bytes 0, 7, 9, 255; non-Latin text), and arrival order is
kept. Presto exposes `ArrayBuffer.length` (3 for a 3-byte buffer), which the stock code relies on.

Check on a unit (read-only): count `push message to waitMsgArray` and
`Unable to parse incoming MMUI message` in `/var/log/running_log/log`.

## Related

- Display Architecture
- SM Service Control (smctl)
- jci-ui/01-architecture-and-boot
