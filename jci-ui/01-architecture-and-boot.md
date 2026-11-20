---
tags: [cmu150, hmi, jci-ui, architecture]
---

# 01 — Architecture & boot

## The stack

The HMI is a **web application rendered by Opera/Presto** (the engine JCI
shipped ~2011-2013). It runs as one page (`index.html`) in a browser surface
that **Wayland / IVI-shell** composites with the other native surfaces
(backup camera, native gauges, test mode). See the display architecture note for the surface/compositor side.

Engine version (fw 74.00.324A, read live from `navigator.userAgent`):
`Opera/9.80 (Linux armv7l) Presto/2.12.362 Version/12.10` — i.e. desktop
Opera 12.10 (2012) behaviour (paint behaviour differs, see below).

Observed engine behaviour on the unit:

- Inline SVG (`<svg><path>`) renders.
- Synthetic `keydown` via `document.createEvent("Events")` + `initEvent` with a
  `keyCode` expando reaches capture-phase `keydown` listeners.
- Scrolling a container by changing `transform: translateY()` leaves stale,
  partially repainted regions (missing text/icons/card edges that persist across
  frames). Desktop Opera 12.10 does not show this. Moving it by `top` instead
  shows no stale regions in knob-focus frames; finger-drag scroll is not established.

Implications that shape everything else:

- **ES5-era JS only.** No `let`/`const` guarantees, no Promises, no `fetch`,
  no modules. Code is `function Foo(){}` + `Foo.prototype.x = function(){}`,
  globals on `window`, callbacks everywhere. WebSocket **is** available (it's
  the entire IPC layer).
- **No DOM framework.** The "framework" here is JCI's own `GuiFramework`, not
  React/Angular. UI is built by instantiating **controls** and **templates**
  that append their own HTML to the DOM.
- The native backend owns application logic and *what screen to show*; the GUI
  is a thin renderer that reacts to messages. "MMUI" = the native multi-modal
  UI manager on the other end of the main socket.

## Web root layout

```
/jci/gui/
├── index.html                 # the single page; fixed script load order
├── framework/
│   ├── js/   GuiFramework.js BaseApp.js Websockets.js Localization.js
│   │         Utility.js Log.js Debug.js
│   └── css/  Reset.css GuiFramework.css Debug.css
├── common/
│   ├── js/        Common.js GuiConfig.js Multicontroller.js
│   │              ContextCategory.js logDefaults.js
│   │              developer_mode_GuiConfig.js   (8-line stub; see below)
│   ├── controls/  Button ClockSettings2 DialPad2 Dialog3 InCall2 Keyboard
│   │              LeftBtn List2 Messaging2 NowPlaying4 Sbn ScreenSettings2
│   │              ScrollDetail Slider StatusBar Tabs Tuner4 Ump3 Wink
│   ├── templates/ ClockSettings2 DialPad2 Dialog3 InCall2 Keyboard List2
│   │              Messaging2 NoCtrl NowPlaying4 ScreenSettings2 ScrollDetail Tuner4
│   └── transitions/ js/Transitions.js js/TransitionsGen.js css/...
├── apps/<uiaId>/              # 42 apps — see [[apps-inventory]]
│   ├── js/<uiaId>App.js  js/<uiaId>AppDict_<lang>.js
│   ├── css/<uiaId>App.css (+ .scss source)
│   ├── images/
│   ├── controls/<Name>/js/<Name>Ctrl.js  css/...
│   └── templates/<Name>/js/<Name>Tmplt.js css/...
└── resources/
    └── js/ GuiResources.js  <uiaId>/<uiaId>AppDict_<lang>.js   # centralized dicts
```

Each **control** lives in `<root>/controls/<Name>/js/<Name>Ctrl.js` (+ css,
+ `_rtl.css` for RTL). Each **template** in `<root>/templates/<Name>/js/<Name>Tmplt.js`.
Controls/templates can be **common** (`common/...`) or **app-specific**
(`apps/<uiaId>/...`). The naming is a hard convention the loader relies on
(see [[04-app-anatomy]]).

## `index.html` — the load chain (ORDER IS LOAD-BEARING)

Scripts, in exact order (the file literally comments "ORDER IS IMPORTANT"):

1. `common/js/GuiConfig.js` — instantiates global **`guiConfig`** (self-constructs)
2. `common/js/logDefaults.js` — the `logDefaults` per-source log-level map
3. `framework/js/Log.js` — instantiates global **`log`**
4. **`../../data/integration/developer_mode_GuiConfig.js`** ← *loaded only if it exists* — **THE INJECTION HOOK**
5. `framework/js/Utility.js`
6. `framework/js/Websockets.js`
7. `resources/js/GuiResources.js` — instantiates global **`guiResources`**
8. `framework/js/BaseApp.js`
9. `framework/js/GuiFramework.js`
10. `framework/js/Localization.js`
11. `framework/js/Debug.js`
12. `common/js/Multicontroller.js`
13. `common/transitions/js/Transitions.js`, `TransitionsGen.js`
14. `common/js/ContextCategory.js`

CSS (also ordered): `Reset.css` → `GuiFramework.css` → `Debug.css` → `TransitionsGen.css`.

Then the **body** runs the bootstrap inline:

```js
var utility   = new Utility();
var baseApp   = new BaseApp();      // used to graft standard fns onto every app
var framework = new GuiFramework();
framework.init();
```

Path note: resolved from `/jci/gui/`, `../../data/integration/...` =
**`/data/integration/developer_mode_GuiConfig.js`**. `/data` is the writable
ffx01p4 partition — that's why the hook works without touching rootfs.

Scripts outside `/data` load too: the page is `file://localhost/jci/gui/index.html`,
and a `<script>` added at runtime with src `../../data_persist/<path>.js` loads from
`/data_persist` (eMMC) and fires `onload`; no error appears in `/var/log/running_log/log`.

## Boot handshake (GUI ⇄ native)

The GUI is dark until a multi-step handshake completes. Sequence:

1. **`framework.init()`** creates `Debug`, **`Websockets`** (opens all 3 sockets),
   **`Localization`**, **`Multicontroller`**, **`Transitions`**; calls
   `_addAppFiles("common")` to load `Common.js`; creates the offscreen template
   parent div.
2. Websockets retries every 500 ms until all three (`mmui`/`appsdk`/`dbapi`)
   reach `readyState==1`, each firing `framework.*Connected(true)`. After 100
   failed retries → **offline mode** (dev/PC only). `debugMode==true` skips
   sockets entirely.
3. Once sockets are up **and** common has finished loading, framework calls
   **`_getStartupSettings()`** → sends event **`common / Global.GetStartupSettings`**
   to MMUI. A 20 s timeout fills in defaults (region NorthAmerica, lang en_US,
   12 hr, Fahrenheit, Miles, VehicleType J36) if `syssettings` is slow.
4. `syssettings` app replies (region/lang/units via setters) → framework calls
   **`initGui()`** → sends event **`common / Global.InitGui`** to MMUI and calls
   `websockets.initGui()` which **flushes** the pre-InitGui message queue.
   (Before InitGui, Websockets only lets through `msgType:"msg"` for
   `uiaId:"syssettings"`; everything else is queued.)
5. Native starts sending real context/focus-stack messages; the first app loads
   and its first template displays. `BaseApp._templateDisplayed` then calls
   **`framework.showOperaSurface()`** and, once only, 1 s later sends event
   **`system / InitializeGuiComplete`** — the signal for NativeGUI to hand over
   all surface management to the HMI. Screen is now live.

So the three GUI→native milestones, in order:
`Global.GetStartupSettings` → `Global.InitGui` → `InitializeGuiComplete`.

## Framework state machine

`GuiFramework._frameworkState` gates message processing. States seen:
`_FWK_STATE_IDLE`, `_FWK_STATE_LOADING_CMN`, `_FWK_STATE_LOADING_APP`,
`_FWK_STATE_LOADING_APP_CTRLS`, `_FWK_STATE_LOADING_TMPLT`.

- Incoming MMUI/DBAPI/AppSDK messages are **only processed in IDLE**; otherwise
  they're pushed to `_msgQueue` and drained by `_processNewFwkState()` when the
  framework returns to idle (see [[02-messaging-bus]]).
- Loading an app/template/control flips the state, loads files async via
  `utility.loadScript`, and each file's trailing `register*Loaded(...)` call
  reports completion; when `_filesToLoad` is all-true, `_*FinishedLoading()`
  advances back toward idle.
- A stuck load trips `_stateTimeout()` → `_showFatalErrorWink` / `_restartCMU`.
  A WebSocket close after a good connection also triggers `_restartCMU` (hard
  GUI restart). Injected code must therefore never block the load chain.

## Page reload reboots the unit — never reload the HMI page

`location.reload()` (or any navigation of the top page) in the running HMI **reboots the whole
unit** about 55–60 s later. Bench-confirmed 2026-11-16, twice:

- bare `location.reload()` at uptime 7950 s, nothing else running: last answer at 8003 s, then the
  board reset (new boot, uptime 73 s at the next check);
- the same ~1 min delay the first time, after a reload with a page-side WebSocket client open.

No core dump, no reset reason in the next boot's log (`running_log` is tmpfs). The ~60 s delay
matches the SM watchdog (`sm.conf`: `watchdog_enable="true" reports_timeout="60000"`); `Opera` is
`reset_board="yes" retry_count="0"`. Likely chain: the reload closes the GUI sockets (MMUI 2700,
AppSDK 2800, DBAPI 2766) → the native side loses the GUI → the GUI stops reporting → SM resets the
board. Exact trigger not pinned.

There is no safe HMI-only restart:
- page reload → board reset ~1 min later (above);
- `smctl -r -n Opera` restarts Opera without a board reset, but it comes back with **no display
  output** (black screen; the IVI/Wayland surface is not re-bound).

So code injected into the HMI page (and any tooling that evals into it) must never reload or
navigate the top page. To pick up a changed script, re-append it with a cache-buster, or SafeReboot.

## A JS exception inside a transition reboots the unit

An uncaught exception in the template-transition path (`Transitions.prototype.*`) leaves the
framework out of IDLE. About 8 s later the GUI framework gives up and the board resets.
Bench log 2026-11-20 (`saved_logs` archive of the crashed boot), all times relative to the exception:

| t | Log line (shortened) |
|---|---|
| 0 | `GUI_FRAMEWORK Error … Uncaught exception: TypeError …` (thrown right after `Transitions.doTemplateTransition called`) |
| +8.0 s | `State Timeout Error: Framework never returned to IDLE state from: templateChange` |
| +8.0 s | `Framework has timed out while not in debug mode. Attempting to restart GUI.` |
| +9.0 s | `FATAL ERROR - RESTARTING CMU : State Timeout` → `GUI_MANAGER Critical (guiManager.c vp_forceExit) CALLING TERMINATE !` |
| +9.0 s | `OPERA_MAIN` exits through `main` (`LOOOP`/`DBUS`/`CLOSE`/`terminate`/`shutdown`, no signal, no core) |
| +10.2 s | `SM … service "Opera" configured as reset_board="yes" exited. Stop pinging watchdog` / `Expect a reboot in up to 5 seconds.` |

- No core dump is written (Opera exits normally). The reason survives the reboot only in
  `/data_persist/log/saved_logs/logsNNN_coreNNN_<ts>-<ver>.tar.bz2`: it holds the
  `running_log` archives of the last ~12 min plus the final `log`.
- In that boot the stock `focusStack` handler also threw (`'this._isStack' is not a function`,
  `Websockets.js:297`) for the same context change. That error alone did not stop the transition
  in earlier boots.
- Consequence for injected code: any wrapper around `Transitions` methods (or any code on the
  transition path) must never throw. A top-level `var` in code evaluated into the page is a
  global, so another script can overwrite a captured original (`var b = P.method` → later
  `b.apply` fails). Keep captured originals in a closure.
