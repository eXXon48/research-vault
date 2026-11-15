---
tags: [cmu150, hmi, custom-apps, gauges, reference]
---

# Custom Apps on the CMU150 HMI

Community "gauges / custom apps" (boost, oil, G-meter, speedo…) are extra HTML5/JS apps
loaded into the stock HMI. Firmware `74.00.324A` (GUI dated Aug 2022) contains the framework
they target. Related: [[jci-ui/README]].

## HMI architecture (JCI-IHU, Johnson Controls, 2011-2012 code)
- `/jci/gui/index.html` loads the framework, then `framework.init()` (GuiFramework.js).
- Rendering engine is **Opera** (`/jci/opera/...`).
- Each screen is an app folder by convention: `apps/<uiaId>/js/<uiaId>App.js` (+css,
  +localization dicts). On load it self-registers via
  `GuiFramework.registerAppLoaded(uiaId, controlList, hasLangSupport)`.
- Stock apps: `fmradio`, `syssettings`, `vehsettings`, `ecoenergy`, `emnavi`,
  `androidauto`, `carplay`, … (see [[jci-ui/apps-inventory]]).

## Extension points in 74.00.324A
Three mechanisms are present and ungated:
1. **Built-in injection hook on `/data`:** `index.html` line 33 loads
   `../../data/integration/developer_mode_GuiConfig.js` *if the file exists*. `/data` is a
   symlink to the persistent data partition (`/mnt/data`). A script at that path runs at HMI
   start. Creating the file needs no rootfs remount; deleting it removes the effect. This is
   the bootstrap point used by the CASDK community tooling.
2. **Generic app loader, no integrity gate:** the loader (GuiFramework.js ~1929-1975) builds
   the path `apps/<uiaId>/js/<uiaId>App.js` and calls `utility.loadScript()`. `framework/` and
   `common/js` contain no signature, whitelist, allowlist, sandbox or checksum check. The
   framework loads any app folder it is told to register.
3. **AppSDK channel:** the HMI ships an Application SDK interface (Websockets.js v0.5 "Added
   AppSDK support"; GuiFramework has `_appsdkReqCallbacks` / `sendAppsdkMsg(...)` RPC
   machinery).

Scope: these are static facts for 74.00.324A. Whether the native backend accepts an AppSDK
WebSocket connection from an injected app is not established statically; everything the app
side needs is present.

## Reachable sockets
HMI ↔ native backend is **three WebSockets** (addresses in `common/js/GuiConfig.js`, `ip` =
local):

| socket | URI | protocol | role |
|---|---|---|---|
| **AppSDK** | `ws://<ip>:2800` | `ihu-appsdk-protocol` | app RPC to native services |
| MMUI | `ws://<ip>:2700` | `ihu-guiifm-protocol` | UI framework bus (focus/context/lifecycle) |
| DBAPI | `ws://<ip>:2766` | `ihu-dbapi-protocol` | settings / DB access |

AppSDK is request/response RPC: the app calls `serviceName.methodName(params)` and gets a
callback (`GuiFramework` `_appsdkReqIdCtr` / `_appsdkReqCallbacks`). An app can subscribe to
the **same backend services the stock apps use**.

**Vehicle data source:** `/jci/vbs/` is **VBS (Vehicle Bus Services)**, with modules
`VBS_CAN`, `VBS_SETTINGS`, `VBS_NAVI`, `VBS_RADIO`, `VBS_DIAG` (vbs.conf). CAN-derived signals
flow VBS → backend services → AppSDK → HMI. The stock `ecoenergy` app pulls live
fuel-consumption / fuel-efficiency data this way, so speed/fuel/rpm-class signals are
available over AppSDK. Exact service and method names appear in the stock apps (e.g.
`ecoenergyApp.js`) and on the live WebSocket. Signals not on the bus (e.g. boost on a turbo
build) are not available from the unit.

The injection hook lives on the persistent `/data` partition, so it touches no rootfs or
`sm.conf`. Community lineage: CASDK / Mazda-AIO.
