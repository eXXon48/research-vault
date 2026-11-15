---
tags: [cmu150, hmi, jci-ui, websocket]
---

# 02 — Messaging bus (MMUI / DBAPI / AppSDK)

All GUI ⇄ native traffic is **three WebSockets**, opened by `Websockets.js`
from `guiConfig`'s socket config. The IP is parsed from `document.URL` (the
http host) or defaults to `127.0.0.1`.

| Socket | URI | WS sub-protocol | Direction & purpose |
|--------|-----|-----------------|---------------------|
| **MMUI** | `ws://<ip>:2700` | `ihu-guiifm-protocol` | The main bus. Native→GUI: what to show (context, focus, data, alerts). GUI→native: user events. |
| **AppSDK** | `ws://<ip>:2800` | `ihu-appsdk-protocol` | Request/response RPC to native services (method calls + responses). |
| **DBAPI** | `ws://<ip>:2766` | `ihu-dbapi-protocol` | Request/response RPC to native DB plugins. |

`<ip>` = regex `http://([^/:]+)` on `document.URL`, else `127.0.0.1`. On target
it's localhost. Each socket auto-reconnects every 500 ms; a close *after* a
successful connect calls `framework._restartCMU(...)` (hard GUI restart).

---

## MMUI — native → GUI (incoming)

`Websockets._mmuiWs.onmessage` → ordering logic → `_doRouteMmuiMsg(jsonStr)` →
`framework.routeMmuiMsg(jsObject)`. `routeMmuiMsg` switches on `msgType`:

| `msgType` | Handler | Meaning / key fields |
|-----------|---------|----------------------|
| `ctxtChg` | `_contextChange(obj)` | Context (screen) change. Fields: `uiaId`, `ctxtId`, `params`, `contextSeq`. If app not loaded → `_addAppFiles(uiaId)` first. `ctxtId === "Idle"` returns early (no screen change); native sends these unprompted for background apps, e.g. `{"uiaId":"contacts","ctxtId":"Idle"}` or `{"uiaId":"ecoenergy","ctxtId":"Idle"}` while another screen stays shown. |
| `focusStack` | `_focusStackChange(appIdList)` | New app focus order. `appIdList` = array of `{id:<uiaId>, ...}`; `appIdList[0].id` is the top app. If top not loaded → load it. Empty/`unknown` list ignored. |
| `transition` | `_transitionChange(enabled)` | `enabled:bool` — brackets a screen transition; GUI defers message processing while a transition is enabled. Duplicate `enabled:false` msgs are de-duped in the queue. |
| `msg` | `_handleMessage(uiaId, msg)` | App-specific data message. Routed to the app's `_messageTable[msg.msgId]`. If app not loaded → load it. |
| `alert` | `common.handleAlert(uiaId, alertId, params)` | Show an alert/wink. If app not loaded → load it. |
| `noGoBack` | `_removeContextRestoreData(contextSeqs)` | Drop saved context-restore data for the given seq numbers. |

**Ordering / sequence counter.** The raw MMUI frame may carry a trailing
sequence byte (`_isTargetMsg` detects `ctxtChg`/`focusStack`/`transition` by
substring). `onmessage` reorders these three types by a wait-seq counter so a
`ctxtChg` and its matching `focusStack`/`transition` are delivered as a set and
in order. After 50 received messages it force-flushes any stuck wait queue.
*You almost never touch this; just know frames can be reordered/queued.*

**Framework-busy queueing.** `routeMmuiMsg` only acts in
`_FWK_STATE_IDLE`; otherwise it pushes `{socketType:"mmui", msg}` to `_msgQueue`
and returns. The queue drains when the framework goes idle again.

**Pre-InitGui gate.** Before `Global.InitGui`, `_doRouteMmuiMsg` forwards only
`msgType:"msg"` with `uiaId:"syssettings"`; all else is queued in
`_preInitGuiMsgQueue` and flushed by `websockets.initGui()`.

### MMUI — GUI → native (outgoing)

Two outbound shapes, both via `Websockets`:

**Event** — the workhorse. `framework.sendEventToMmui(uiaId, eventId, params, fromVui)`
→ `websockets.sendEventMsg(...)` → sends:

```json
{ "msgType":"event", "eventId":"<Event>", "uiaId":"<uiaId>",
  "params":{...}, "fromVui":false,
  "currentUiaId":"<auto>", "currentContextId":"<auto>" }
```

`currentUiaId`/`currentContextId` are auto-filled from
`framework.getCurrentApp()`/`getCurrCtxtId()` (the in-file comment warns they're
not always accurate). `params` defaults to `"{}"`. Event IDs are a native-side
contract; common ones seen: `Global.GetStartupSettings`, `Global.InitGui`,
`Global.GoBack`, `Global.Yes`, `Global.No`, `Global.Cancel`,
`InitializeGuiComplete`, and per-menu `Select*` (e.g. `SelectRadioFM`,
`SelectBTAudio`, `SelectEntertainmentMenu`). See [[04-app-anatomy]] for how menu
items map to events.

**Alert-complete** — `websockets.sendAlertCompleteMsg(uiaId, alertId, params)`:

```json
{ "msgType":"alertComplete", "alertId":"<id>", "uiaId":"<uiaId>", "params":{...} }
```

---

## AppSDK — request/response RPC

**Send:** `framework.sendRequestToAppsdk(uiaId, callbackFn, serviceName, methodName, params)`.
Framework increments `_appsdkReqIdCtr`, stores `callbackFn` under key
`serviceName+methodName+reqId` (expires after `_DATA_REQUEST_MAX_AGE`, GC'd by
`_dataRequestGarbageCollector`), then `websockets.sendAppsdkMsg(...)` sends:

```json
{ "msgType":"method", "serviceName":"<svc>", "methodName":"<m>",
  "reqId":<n>, "params":{...} }
```

**Receive:** `routeAppsdkMsg` matches `msgType:"methodResponse"` *or*
`"methodErrorResponse"` by `serviceName+methodName+reqId`, invokes the stored
callback with the full response object, then deletes it.

## DBAPI — request/response RPC

**Send:** `framework.sendRequestToDbapi(uiaId, callbackFn, api, methodName, params)`.
Same callback-table pattern, keyed `api+methodName+reqId`. Sends:

```json
{ "msgType":"method",
  "msgContent": { "api":"<plugin>", "methodName":"<m>", "reqId":<n>, "params":{...} } }
```

**Receive:** `routeDbapiMsg` matches `msgType:"methodResponse"`, keyed
`msgContent.api + msgContent.methodName + msgContent.reqId`, fires + deletes the
callback.

Both AppSDK and DBAPI responses are also subject to the IDLE-state queueing.

