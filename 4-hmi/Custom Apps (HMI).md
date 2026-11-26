---
tags: [cmu150, hmi, custom-apps, gauges, reference]
---

# Custom Apps (HMI)

Where extra HTML5/JS apps (gauges, meters and similar) can hook into the stock HMI, which sockets
they can reach, and where vehicle data comes from.

## Overview

The stock HMI is an Opera-based HTML/JS framework (JCI-IHU, code from 2011–2012; the GUI in
firmware `74.00.324A` is dated August 2022). It has three extension points, none of them gated:

1. a script hook on writable `/data`, loaded at HMI start if the file exists;
2. a generic app loader with no integrity check;
3. the AppSDK WebSocket channel to native services.

Vehicle signals reach apps through VBS, the backend services and AppSDK.

Related: [[4-hmi/jci-ui/README|HMI framework reference]], [[apps-inventory]],
[[Vehicle Data (VBS)]], [[HMI Performance Levers]].

## Reference

### HMI structure

- `/jci/gui/index.html` loads the framework, then calls `framework.init()` (`GuiFramework.js`).
- The rendering engine is Opera (`/jci/opera/...`).
- Each screen is an app folder: `apps/<uiaId>/js/<uiaId>App.js`, plus CSS and localization
  dictionaries. On load the app registers itself with
  `GuiFramework.registerAppLoaded(uiaId, controlList, hasLangSupport)`.
- Stock apps include `fmradio`, `syssettings`, `vehsettings`, `ecoenergy`, `emnavi`,
  `androidauto` and `carplay` ([[apps-inventory]]).

### Extension points

| Mechanism | Where | Notes |
|---|---|---|
| Injection hook | `index.html` line 33 loads `../../data/integration/developer_mode_GuiConfig.js` if the file exists | `/data` is a link to the persistent data partition (`/mnt/data`). The script runs at HMI start. Creating it needs no rootfs remount; deleting it removes the effect. |
| App loader | `GuiFramework.js` ~1929–1975 builds `apps/<uiaId>/js/<uiaId>App.js` and calls `utility.loadScript()` | `framework/` and `common/js` have no signature, allowlist, sandbox or checksum check. The framework loads any app folder it is told to register. |
| AppSDK channel | `Websockets.js` v0.5 ("Added AppSDK support"); `GuiFramework` `_appsdkReqCallbacks`, `sendAppsdkMsg(...)` | request/response RPC to native services |

The injection hook touches neither the rootfs nor `sm.conf`.

### Sockets

The HMI talks to the native backend over three WebSockets. Addresses are in
`common/js/GuiConfig.js` (`ip` = local).

| Socket | URI | Protocol | Role |
|---|---|---|---|
| AppSDK | `ws://<ip>:2800` | `ihu-appsdk-protocol` | app RPC to native services |
| MMUI | `ws://<ip>:2700` | `ihu-guiifm-protocol` | UI framework bus (focus, context, lifecycle) |
| DBAPI | `ws://<ip>:2766` | `ihu-dbapi-protocol` | settings and database access |

AppSDK is request/response RPC: the app calls `serviceName.methodName(params)` and gets a callback
(`GuiFramework` `_appsdkReqIdCtr` / `_appsdkReqCallbacks`). An app can use the same backend services
the stock apps use.

### Vehicle data

- `/jci/vbs/` is VBS (Vehicle Bus Services), with modules `VBS_CAN`, `VBS_SETTINGS`, `VBS_NAVI`,
  `VBS_RADIO` and `VBS_DIAG` (`vbs.conf`).
- CAN-derived signals flow VBS → backend services → AppSDK → HMI. The stock `ecoenergy` app reads
  live fuel consumption and fuel efficiency this way, so speed, fuel and rpm-class signals are
  available over AppSDK.
- Exact service and method names are in the stock apps (for example `ecoenergyApp.js`) and on the
  live WebSocket.
- Signals that are not on the bus (for example boost on a turbo engine) are not available from the
  unit.

## Verified on

Stock rootfs of firmware `cmu150_EU_74.00.324A` (static analysis).

## Open questions

- Whether the native backend accepts an AppSDK WebSocket connection from an injected app. The app
  side has everything it needs.
