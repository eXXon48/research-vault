---
tags: [cmu150, hmi, jci-ui, reference]
---

# 05 — Framework modules (per-module API)

Reference tables for the non-`GuiFramework` modules. All reachable via globals or
`framework.*`.

---

## GuiConfig  (`guiConfig`, common/js/GuiConfig.js)

Config flags (all should be **false** in a shipped build):

| Flag | Effect |
|------|--------|
| `debugMode` | Skip WebSockets; load `*Test.js` fake-MMUI files; offline. |
| `testMode` | Load JasmineManager auto-test framework. |
| `promptForIp` | Prompt for target IP on load (PC-to-target dev). |
| `debugPanelEnabled` | Right-side debug panel + message log. |
| `screenNameLabelEnabled` | Show current context name on screen. |
| `disableLogs` | Kill all console logs. |
| `showFrameworkErrorAlerts` | Show a wink on framework state-timeout errors. |
| `pcLogging` | PC vs target log formatting. |
| `ignoreDictsFailure` (true) | Don't error if some dicts fail to load. |
| `showStringIdFailure` | Show `lang:` prefix on missing string ids. |
| `winkTimeout` (3000) | ms a wink stays on screen. |

Socket config built in constructor: `appsdkSocket`/`mmuiSocket`/`dbapiSocket`
= `{uri, protocol}` (ports 2800/2700/2766). IP from `document.URL` or `127.0.0.1`.

**`guiConfig._setDeveloperMode(flag)`** — toggles `screenNameLabelEnabled`,
`showFrameworkErrorAlerts`, `showStringIdFailure` together. This is what the stock
`developer_mode_GuiConfig.js` stub calls. (The injection hook reuses that file's
*path* but need not call this.)

---

## Utility  (`utility`, framework/js/Utility.js)

| Method | Purpose |
|--------|---------|
| `loadScript(url, attributes, callback)` | **Async-load a JS file** into the DOM (the loader's workhorse). |
| `removeScript(url)` / `loadCss(url)` / `removeCss(url)` | Script/CSS add-remove. |
| `emptyHTMLContent(elementId)` / `removeHTMLElement(idOrObj)` | DOM clearing. |
| `sanitizeHtml(str, convertEscapeChars)` | Escape untrusted text. |
| `toType(obj)` | Robust type string (`'array'`, `'object'`, …). |
| `getArrayItemByPropertyValue(array, property, value)` | Find in array. |
| `deepCopy(p, c)` | Deep clone (used for context capture). |
| `hideSurface(id,time)` / `showSurface(id,time)` / `moveSurface(id,x,y)` / `setInputSurface(id)` / `setRequiredSurfaces(surfaces, fadeOpera)` / `hideAllSurfaces()` | **Wayland surface control.** |
| `formatTime(sec, showSeconds)` / `formatDate(sec)` / `formatDateTime(...)` / `formatSmartDateTime(...)` / `findTimeDuration(d1,d2)` | Locale-aware date/time. |
| `getDistanceUnitLabel / getTemperatureUnitLabel / getVolumeUnitLabel(dontLocalize)` | Unit labels. |

---

## Localization  (`framework.localize`, framework/js/Localization.js)

Dictionary + locale manager.

**Strings:** `getLocStr(defaultAppName, stringId, subMap)`,
`testLocStr(app, id)`, `getVrHelpStringId(...)`, `loadAppDict(uiaId)`,
`addLangSupport(uiaId)`, `unloadDictionaries(lang, status)`, `getLoadedDicts()`,
`getAppsWithLangSupport()`.

**Locale getters/setters** (each setter takes an enum key; validated against an
enum object): `setLanguage(enumKey, vrSupport, reloadAll, callbackFn)` /
`getCurrentLanguage()`, `setKeyboardLanguage`/`getKeyboardLanguage`,
`getPreviousLanguage`, `setRegion`/`getRegion`, `setTimeFormat`/`getTimeFormat`,
`setDateFormat`/`getDateFormat`, `setVolumeUnit`/`getVolumeUnit`,
`setTemperatureUnit`/`getTemperatureUnit`, `setDistanceUnit`/`getDistanceUnit`,
`setDisplayTheme`/`getDisplayTheme`, `setVehicleType(type)`.

Enum tables on the object: `REGIONS`, `TIME_FORMATS`, `TMPRTURE_UNITS`,
`DISTANCE_UNITS`, etc. (used by the startup default-fill). Format converters:
`getLangInMmuiFormat(enumKey)`, `getLangInGuiFormat(enumKey)`.

These are driven at boot by `syssettings` responding to `Global.GetStartupSettings`.

---

## Multicontroller  (`framework.multicontroller`, common/js/Multicontroller.js)

Translates keyboard/rotary/touch into **tuiEvents** delivered to the current
template via `framework._multicontrollerEventHandler(tuiEvent, firstMultiEvent)`.

Key → tuiEvent map (`_keyDownHandler`):

| Key (keycode) | tuiEvent |
|---------------|----------|
| Enter (13) | `selectStart` |
| ← (37) | `leftStart` |
| → (39) | `rightStart` |
| ↑ (38) | `upStart` |
| ↓ (40) | `downStart` |
| `M` (77) | `cw` (rotary clockwise) |
| `N` (78) | `ccw` (rotary counter-clockwise) |
| Backspace (8) | Go Back (preventDefault; no native back) |
| Shift (16) | ignored |

Modes: `touchActive` vs `controllerActive` (`_checkMode`, `getMode`). First
controller event after switching from touch is flagged `firstMultiEvent` and
discarded. Also handles mouse/click/wheel/touch (`_registerTouchActive`,
`_mouseUpEventHandler`, `_clickEventHandler`, `_mouseWheelHandler`) and
`_simulateKeyPress(keycode, shift)` (calls `_keyDownHandler` then `_keyUpHandler`
directly with a plain object; it creates no DOM event, so `window`/`document`
listeners never see it). Every real event also calls
`framework.common.activityDetected(...)` (auto-hide status bar, etc.).

> Physical dash keys arrive as uinput events at the native layer, not here. Multicontroller only sees
> what Presto delivers as DOM key/mouse/touch events.

Listener registration (`Multicontroller.js:49-66`): `mousedown`, `mouseup`,
`click`, `keydown`, `keyup`, `mousewheel` — all on **`document`, capture phase**,
registered at framework start. Consequences:

- Any later `document` capture listener runs **after** the stock one, so its
  `stopPropagation`/`stopImmediatePropagation` cannot keep a key from the stock
  handler. A `window` capture listener runs **before** it (window precedes
  document in the Presto 2.12 event path), and `stopPropagation` there
  hides the key from Multicontroller.
- The stock `_keyDownHandler` latches `_btnHeld` on keydown and clears it on
  keyup; hiding both halves of a press keeps that latch consistent.
- The stock list acts on the keyup of Enter: a lone synthetic `keydown` 13 with no
  keyup selects nothing. The `List2` control's focused row index
  is `list2Ctrl.focussedItem` (double s).
- A DOM layer drawn over the current template does not stop the template from
  receiving commander events: the hidden template keeps moving its own focus and
  acts on `selectStart` (e.g. the Applications list opens whichever row it has
  focused, `Recv ctxtChg:VehicleStatusMonitor`).

Native side of the same press (seen in `/var/log/running_log/log`): MMUI logs
every commander/hard-key press as `FIM_MANAGER_ButtonEventPreFunction
PreProcessing button (<id>)"<name>"` with the current `appId.contextId` — ids
seen: 50 `Commander CW`, 49 `Commander CCW`, 28 `Select`, 103 `Up`, 108 `Down`,
105 `Left`, 14 `Back`. `Back` (14) is handled natively (`GoBack_Evt` → MMUI
context change) independent of the GUI. Tilt Up/Down are additionally forwarded
to the Android Auto service (`mmui_androidauto_TiltUp/TiltDown` → `BLM_AAPA
InputKey keycode[5]/[6]`) even when AA video is not in focus
(`IsAAVideoInFocus = false`).

Commander hard buttons around the knob: Home, Entertainment (media), Favorites, Map/Nav, Back.
The GUI keyboard map has only the knob keys and Backspace (8). It has no keycode for Home,
Entertainment, Favorites or Map. Those buttons are MMUI intents (`Global.IntentHome`,
`Global.IntentActivateFavorites`, plus `IntentCarPlay*` / `IntentAndroidAuto*` variants in
`mmui/uia/libjciuiacommon.so`). MMUI turns them into a context change, which the GUI only
receives (`framework._contextChange`). A GUI script cannot cancel such a press. The
status-bar Home icon is the one GUI-side Home: `Common._statusBarClicked` sends
`common Global.IntentHome`, and a long press sends `Global.IntentHomeLongPress`.

---

## Common  (`framework.common`, common/js/Common.js — 2753 lines)

The shared chrome + cross-cutting behaviour singleton. Highlights (not
exhaustive — grep the file for the rest):

**Status bar / name / icons:** `setSbName(label)`, `setSbNameId(uiaId, labelId, subMap)`,
`setSbDomainIcon(path)`, `setSbIcon(name, visible, state)`, `showSbHomeButton(show)`,
`requestStatusBarShown(delay,dur,cb)`, `requestStatusBarHidden(...)`,
`updateSbClock(ms)`, `getCurrentTime()`, `setLineNumbers(count, style)`.

**Status-bar notifications (SBN):** `startTimedSbn(uiaId,sbnId,type,props)`,
`cancelTimedSbn(...)`, `showStateSbn(...)`, `endStateSbn(...)`,
`_requestNewSbn(...)`, `_displaySbn(id)`.

**Alerts / winks:** `handleAlert(uiaId, alertId, params)`, `_alertComplete(...)`,
`_removeWinkHelper()`.

**Controller/data plumbing:** `handleControllerEvent(eventId, firstEvent)`,
`handleDataMessage(msg)`, `stealFocus()`, plus VUI handlers
(`_ShowVuiHelpHandler`, `_ShowVuiExamplesHandler`, …), speed lock
(`_AtSpeedMsgHandler`/`_NoSpeedMsgHandler`/`getAtSpeedValue()`), page up/down,
language-change status.

**Activity / inactivity:** `activityDetected(isActive, evt, tuiEvent)`,
`_startInactivityTimer()`, `_onInactivity()`.

**Mute / audio feedback:** `beep(pressType, eventCause)`, `_showMuteSbn()`,
`_showMuteWink(...)`, `_HandleStatusUpdateVolume(msg)`.

**Controls lifecycle:** `addControls()`, `removeControls()`,
`getCommonTransitionData(template)`, `commonControlsUpdateComplete(...)`,
`getContextCapture()`, `restoreContext(restoreData)`.

Common also loads its own controls (status bar, left button, winks, SBNs) at
startup via `registerCommonLoaded`.

---

## ContextCategory  (common/js/ContextCategory.js)

`getContextCategory(uiaId, ctxtId)` → domain string; lookup table keyed
`"<uiaId>.<ctxtId>"` then `"<uiaId>.*"`, default `"Other"`. Domains:
`Applications, Communication, Entertainment, Settings, Navigation, Favorites,
Other`. `getContextCategorySbIcon(domain)` → icon filename
(`IcnSbnApps.png`, `IcnSbnComm.png`, `IcnSbnEnt.png`, `IcnSbnFavorite.png`,
`IcnSbnMaint.png`, `IcnSbnMap.png`, `IcnSbnSettings.png`, `""`).

---

## GuiResources  (`guiResources`, resources/js/GuiResources.js)

Manifests only (no logic):
- **`DICTIONARY_LIST`** — `{uiaId: bool}`; truthy ⇒ dictionaries are centralized
  under `resources/js/<uiaId>/`. 44 entries; `screenrep:false` is the lone false.
- **`VR_EXAMPLES`** — `{examplesId: {titleText, textId1..5}}` for voice-help
  example screens.

---

## Log  (`log`, framework/js/Log.js)

`log.debug/info/warn/error/fatal(text)`, `writeError(err)`. Sources register via
`log.addSrcFile(fileName, sourceModule)` (first line of most files). Per-source
gates default from `logDefaults` (`common/js/logDefaults.js`): `__global__:"info"`
is the master gate; levels `fatal>error>warn>info>debug`. Control at runtime:
`setLogLevel(source, level)`, `squelch(source)`, `solo(source)`,
`restoreDefaultLevels()`.
