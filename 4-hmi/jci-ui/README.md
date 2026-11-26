---
tags: [cmu150, hmi, jci-ui, moc]
---

# HMI Framework Reference

**What this is:** a reference for how the Mazda CMU150
(JCI / Johnson Controls "IHU") head-unit **HMI GUI** works: the paths, globals,
functions, message names and config keys of the UI, and its extension points.

**Source of truth:** the stock firmware's `/jci/gui/**` web root. Paths, globals,
function names, message formats and config keys are taken from the stock JS
files; line numbers refer to those stock files.

**The one-line summary:** the HMI is an **Opera/Presto (≈2012) web app** at
`/jci/gui`, driven by a native backend over **three WebSockets** (MMUI/DBAPI/
AppSDK). A tiny JS framework (`GuiFramework`) loads per-screen **apps**
on demand by predictable path, each app a JS object with a context table and a
message table. Native decides *what* screen shows (context/focus-stack
messages); the GUI renders it from templates + controls. The injection point is
a **stock loader line** that runs arbitrary JS from the writable `/data`
partition — no rootfs edit required.

---

## Table of contents

| # | Doc | Covers |
|---|-----|--------|
| 01 | [[01-architecture-and-boot]] | Opera/Presto + Wayland, web root layout, `index.html` load chain, boot handshake, framework state machine |
| 02 | [[02-messaging-bus]] | The 3 sockets (ports/protocols), every message format, routing, send APIs, startup events |
| 03 | [[03-framework-api]] | `GuiFramework` full public API + the global objects (`framework`, `guiConfig`, `log`, `utility`, `baseApp`, `guiResources`) |
| 04 | [[04-app-anatomy]] | App directory layout, app skeleton, `BaseApp` lifecycle, `_contextTable`/`_messageTable`, the **menu dataList schema**, controls & templates, shared data, localization |
| 05 | [[05-framework-modules]] | Per-module API tables: Utility, Localization, Multicontroller, Common, GuiConfig, ContextCategory, GuiResources, Log |
| — | [[apps-inventory]] | All 42 apps + their context-category domains |

## Fast facts (cheat sheet)

- **Web root:** `/jci/gui/` → `index.html`, `framework/`, `common/`, `apps/`, `resources/`.
- **Engine:** Opera/Presto. ES5-era JS, no modern APIs. Surfaces composited by Wayland/IVI-shell.
- **Canvas 2D quirk:** Presto 2.12 (Opera 12.10) `CanvasRenderingContext2D.arcTo()` renders wrong (rounded rects come out as petal/blob shapes). `arc()` + `lineTo()` render correctly; build rounded corners from `arc()`.
- **Canvas 2D on the unit:** an 800x480 `<canvas>` with `fillRect`, `arc`, `lineTo`/`stroke`, `createLinearGradient`, `globalAlpha` and `fillText` renders correctly in the HMI page; `setTimeout` animation at ~33 ms runs (no `requestAnimationFrame`).
- **Inline SVG:** SVG markup set via `innerHTML` (`<svg viewBox>` + `<path>` with `fill`/`stroke`) renders in the HMI page; usable for icons where the HMI font lacks a glyph (e.g. U+2605).
- **Images from writable storage:** `<img src="/data_persist/...">` (absolute filesystem path) loads in the HMI page (page origin is `file://`).
- **SafeReboot:** the dbus `com.jci.blm.system` SafeReboot call answers `int32 100` when SM is up; the unit is reachable over SSH again ~110 s later.
- **Sockets:** MMUI `ws://127.0.0.1:2700` (`ihu-guiifm-protocol`), AppSDK `:2800` (`ihu-appsdk-protocol`), DBAPI `:2766` (`ihu-dbapi-protocol`).
- **Globals (window):** `guiConfig`, `log`, `utility`, `baseApp`, `framework`, `guiResources`; on `framework`: `.websockets`, `.localize`, `.multicontroller`, `.transitionsObj`, `.common`, `.debug`.
- **App path:** `apps/<uiaId>/js/<uiaId>App.js` (+ `css/`, `images/`, `controls/`, `templates/`, `js/<uiaId>AppDict_<lang>.js`). 42 apps.
- **Load an app:** native sends a `focusStack`/`ctxtChg`/`msg`/`alert` for a `uiaId` → `framework._addAppFiles(uiaId)` → app JS ends with `framework.registerAppLoaded(uiaId, controlList, hasLangSupport)`.
- **GUI→native:** `framework.sendEventToMmui(uiaId, eventId, params)`.
- **Injection hook:** `index.html` stock-loads `../../data/integration/developer_mode_GuiConfig.js` *if it exists* → arbitrary JS from writable `/data`.

## Related

Companion CMU150 notes: [[Audio Architecture]],
[[Display Architecture]], [[Custom Apps (HMI)]].
