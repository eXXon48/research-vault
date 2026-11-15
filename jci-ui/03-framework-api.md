---
tags: [cmu150, hmi, jci-ui, api]
---

# 03 — Framework API & globals

## Global objects (on `window`)

| Global | Class / file | What it is |
|--------|--------------|-----------|
| `guiConfig` | `GuiConfig` (common/js/GuiConfig.js) | Config flags + the 3 socket definitions. Self-instantiates. See [[05-framework-modules]]. |
| `log` | `Log` (framework/js/Log.js) | Logger. `log.debug/info/warn/error/fatal(text)`. |
| `utility` | `Utility` (framework/js/Utility.js) | DOM/script/CSS/format helpers. |
| `baseApp` | `BaseApp` (framework/js/BaseApp.js) | Grafts standard fns+vars onto every app via `baseApp.init(this, uiaId)`. |
| `framework` | `GuiFramework` (framework/js/GuiFramework.js) | **The orchestrator.** Everything below hangs off it. |
| `guiResources` | `GuiResources` (resources/js/GuiResources.js) | `DICTIONARY_LIST`, `VR_EXAMPLES` manifests. |

Sub-objects on `framework` (created in `init()`):
`framework.websockets`, `.localize`, `.multicontroller`, `.transitionsObj`,
`.debug`, and `.common` (set during the `Common` constructor). `framework.common`
is the `Common` singleton — status bar, alerts, SBNs, mute, beep, etc.

---

## GuiFramework — public API

Grouped by purpose. Signatures verbatim from the prototype.

### Lifecycle / startup
| Method | Notes |
|--------|-------|
| `init()` | Called once from `index.html`. Builds submodules, loads common. |
| `initGui()` | Sends `common/Global.InitGui`, flushes WS queue. Called after startup settings arrive. |
| `_getStartupSettings()` | Sends `common/Global.GetStartupSettings`; 20 s default-fill timeout. |
| `_heartbeat()` | Periodic liveness. |
| `_restartCMU(reason)` | Hard GUI restart (also fired on WS crash). |
| `_showFatalErrorWink(text1, text2)` | On-screen fatal error. |
| `_yieldCpuToOpera()` | Cooperative CPU yield to Presto. |

### Socket connect callbacks (from Websockets)
`mmuiConnected(connected)`, `appSdkConnected(connected)`, `dbApiConnected(connected)`.

### Outbound messaging  → see [[02-messaging-bus]]
| Method | Sends |
|--------|-------|
| `sendEventToMmui(uiaId, eventId, params, fromVui)` | MMUI `event` |
| `sendRequestToDbapi(uiaId, callbackFn, api, methodName, params)` | DBAPI `method` (+ stores callback) |
| `sendRequestToAppsdk(uiaId, callbackFn, serviceName, methodName, params)` | AppSDK `method` (+ stores callback) |
| `sendMsgToFocusedApp(msg)` | Routes a msg to the in-focus app. |

### Inbound routing
`routeMmuiMsg(jsObject)`, `routeDbapiMsg(jsObject)`, `routeAppsdkMsg(jsObject)`,
`_handleMessage(uiaId, msg)`, `_multicontrollerEventHandler(tuiEvent, firstMultiEvent)`.

### Context / focus / transitions
`_contextChange(jsObject)`, `_focusStackChange(appIdList)`,
`_transitionChange(enabled)`, `_callHandleContext(contextObj)`,
`_captureContext(templateInstance)`, `_getContextRestoreData(context)`,
`_removeContextRestoreData(contextSeqs)`, `_shouldUseSnapshot(tempTemplate)`,
`isCurrentTemplateDialog()`, `getCurrCtxtId()`, `getCurrCtxtSeqNo()`.

### App registry / current-app accessors
| Method | Returns |
|--------|---------|
| `getCurrentApp()` | uiaId of the in-focus app |
| `getCurrAppInstance()` | the in-focus app object |
| `getAppInstance(uiaId)` | a loaded app object by id |
| `_initApp(uiaId)` | instantiate an app |
| `_storeAppLoaded(uiaId)` | mark app present in `_appStack` |

### Dynamic file loading (apps / templates / controls) → see [[04-app-anatomy]]
| Method | Role |
|--------|------|
| `_addAppFiles(uiaId, mmuiMsgObj)` | Load `apps/<uiaId>/js/<uiaId>App.js` (+css). `common` special-cased. |
| `_addTmpltFiles(templateName, path)` | Load a template's js+css. |
| `registerCommonLoaded(controlList, hasLangSupport)` | Trailing call in `Common.js`. |
| `registerAppLoaded(uiaId, controlList, hasLangSupport)` | **Trailing call in every `<uiaId>App.js`.** Triggers control + dict loads. |
| `registerTmpltLoaded(templateName, controlList)` | Trailing call in a template js. |
| `registerCtrlLoaded(controlName)` | Trailing call in a control js. |
| `registerAppDictLoaded(uiaId, langCode)` | Trailing call in a dictionary js. |
| `_commonFinishedLoading()` / `_appFinishedLoading()` / `_tmpltFinishedLoading()` | State advance when `_filesToLoad` all true. |
| `_processNewFwkState(state)` / `_stateTimeout(err)` | State machine + stuck-load timeout. |

### Instantiate / destroy UI
| Method | Notes |
|--------|-------|
| `instantiateControl(uiaId, parentDiv, name, properties, persistentName)` | `new window[name](uiaId, parentDiv, controlID, properties)`; control appends itself to DOM. |
| `instantiateTemplate(uiaId, parentDiv, name, properties, path)` | If not yet loaded → `_addTmpltFiles` + return null (async retry). Else `new window[name](...)`. |
| `destroyControl(controlObj)` | `controlObj.cleanUp()` + remove from DOM. |

### Surfaces / display / transitions
`showOperaSurface()`, `_showTemplateSurfaces(someTemplate, prevTemplate)`,
`_hideTemplateSurfaces(someTemplate)`, `_templateTransitionCompleteCallback()`,
`initGui()` (above), `_createLoadingScreen(displayText)`.

### Status bar
`_setStatusBar(uiaId, ctxId, dataObj)`, `_resetStatusBar()`,
`_isStatusBarDataValid(uiaId, ctxId, dataObj)`. (Most status-bar behaviour is in
`Common` — see [[05-framework-modules]].)

### Language / locale
`changeLanguage(languageCode, callbackFn)`, `_setLanguageSpecificStyling(languageCode)`,
`_dictsLoaded(status)`, `getInputEnabled()`. (Locale getters/setters live on
`framework.localize` — see [[05-framework-modules]].)

### Shared data (cross-app key/value with subscriptions)
| Method | Notes |
|--------|-------|
| `setSharedData(uiaId, name, value)` | Set + notify current app & subscribers via `handleSharedDataChanged`. |
| `getSharedData(uiaId, name)` | Get (or `undefined`). |
| `subscribeToSharedData(uiaId, sharedDataUiaId, sharedDataName)` | Subscribe app `uiaId` to a value owned by `sharedDataUiaId`. |
| `unsubscribeToSharedData(uiaId, sharedDataUiaId, sharedDataName)` | Remove subscription. |

Storage key is `ownerUiaId + name`; value object is `{value, subscribers:[]}`.
On change, the **in-focus** app always gets `handleSharedDataChanged`, plus every
subscriber except the setter itself. Startup example: `syssettings`/`VehicleType`.

---

## Example calls from injected JS

All globals are reachable from code loaded via the injection hook:

```js
// fire an MMUI event as if the user chose FM radio:
framework.sendEventToMmui("system", "SelectRadioFM");

// ask a DBAPI plugin something:
framework.sendRequestToDbapi("system", function(resp){ log.info(JSON.stringify(resp)); },
                             "<api>", "<method>", {});

// read/stash cross-app state:
framework.setSharedData("myapp", "lastInstall", "ok");
var v = framework.getSharedData("myapp", "lastInstall");

// who/where am I:
framework.getCurrentApp(); framework.getCurrCtxtId();
```
