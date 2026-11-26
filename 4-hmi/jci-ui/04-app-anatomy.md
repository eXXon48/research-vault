---
tags: [cmu150, hmi, jci-ui, apps]
---

# 04 — App anatomy & modification surface

An **app** is one `uiaId` (e.g. `system`, `fmradio`, `syssettings`) = one JS
object that renders a family of related screens (**contexts**). Native decides
which app/context is in focus; the app renders it.

## Directory layout (hard convention — the loader depends on it)

```
apps/<uiaId>/
├── js/
│   ├── <uiaId>App.js                 # the app object; MUST end with registerAppLoaded(...)
│   └── <uiaId>AppDict_<lang>.js      # per-language dictionary (unless centralized in resources/)
├── css/
│   ├── <uiaId>App.css  (+ .scss)     # loaded automatically with the app
│   └── <uiaId>App_rtl.css            # loaded additionally when framework._rtlLanguage
├── images/
├── controls/<Name>/js/<Name>Ctrl.js  (+ css/, + _rtl.css)   # app-specific controls
└── templates/<Name>/js/<Name>Tmplt.js (+ css/)              # app-specific templates
```

`_addAppFiles(uiaId)` computes these paths literally: `apps/<uiaId>/js/<uiaId>App.js`,
`apps/<uiaId>/css/<uiaId>App.css`. There is **no registry** — presence at the
predictable path + the trailing `registerAppLoaded` call is the whole protocol.
That's also why adding an app means adding a directory *and* making native send a
focus/context message for it.

## App load sequence

1. Native sends `focusStack`/`ctxtChg`/`msg`/`alert` naming `uiaId`.
2. `routeMmuiMsg` sees the app isn't in `_appStack` → `_addAppFiles(uiaId)` →
   loads css then `utility.loadScript("apps/<uiaId>/js/<uiaId>App.js")`.
3. The app file defines `function <uiaId>App(uiaId){ baseApp.init(this,uiaId); }`
   + prototype methods, then its **last line**:
   `framework.registerAppLoaded("<uiaId>", controlList, hasLangSupport)`.
4. `registerAppLoaded`:
   - marks the app file loaded;
   - for each path in `controlList`, loads `<path>/js/<base>Ctrl.js` (+css);
   - loads the dictionary: if `guiResources.DICTIONARY_LIST[uiaId]` is truthy →
     `resources/js/<uiaId>/<uiaId>AppDict_<lang>.js`; else if `hasLangSupport`
     → `apps/<uiaId>/js/<uiaId>AppDict_<lang>.js`;
   - when `_filesToLoad` is all true → `_appFinishedLoading()` → `_initApp` →
     app instantiated, `appInit()` called, templates/controls instantiated for
     the focused context.

## The app skeleton (from `systemApp.js`)

```js
log.addSrcFile("systemApp.js", "system");       // register log source

function systemApp(uiaId) {
    baseApp.init(this, uiaId);                   // graft standard vars+fns (BaseApp)
    // feature init happens in appInit()
}

systemApp.prototype.appInit = function() {        // called by framework just after instantiation
    this._initEntertainmentDataList();            // build menu data arrays
    this._contextTable = { /* ctxtId -> screen spec */ };
    this._messageTable = { /* msgId -> handler    */ };
    // ... app-specific state ...
};

// ... prototype methods: callbacks, context in/out fns, message handlers ...

framework.registerAppLoaded("system", /*controlList*/ [...], /*hasLangSupport*/ true);
```

## BaseApp — what every app gets for free

`baseApp.init(appInstance, uiaId)` grafts:

**Variables:** `uiaId`, `sbNameIcon`, `_currentContext`, `_currentContextTemplate`,
`_outgoingContext`, `_outgoingContextTemplate`, `_previousContext`,
`_contextChanged`.

**Functions** (bound to the app instance):
`handleContext`, `handleDataMessage`, `isContextDifferent`, `isContextDifferentFrom`,
`hasContextChanged`, `getContextTableEntry`, `templateReadyToDisplay`,
`templateDisplayed`, `templateNoLongerDisplayed`, `getCurrentContext`,
`handleSharedDataChanged`.

**Template lifecycle callbacks** (framework calls these; app hooks them via the
context table): `readyFunction` (template built, not yet on screen) →
`displayedFunction` (finished transitioning on) → `noLongerDisplayedFunction`
(transitioned off, about to be destroyed). Plus `contextInFunction` /
`contextOutFunction` around a context becoming/leaving current.

> **Boot-critical detail:** the very first `templateDisplayed` triggers
> `framework.showOperaSurface()` and, 1 s later and once only,
> `sendEventToMmui("system","InitializeGuiComplete")`. Don't break the first
> template's display path.

## `_contextTable` — the screen spec (per ctxtId)

Each key is a `ctxtId`; the value configures the screen. Fields seen in
`systemApp`:

```js
"HomeScreen" : {
    "hideHomeBtn": true,
    "template": "MainMenuTmplt",
    "templatePath": "apps/system/templates/MainMenu",   // only for app-specific templates
    "controlProperties": {
        "MainMenuCtrl": { "selectCallback": this._selectCallbackHomeScreen.bind(this) }
    }
},
"Entertainment" : {
    "leftBtnStyle": "goBack",          // left button behaviour
    "sbNameId": "Entertainment",       // status-bar name (dictionary id)
    "template": "List2Tmplt",          // a COMMON template (no templatePath needed)
    "controlProperties": {
        "List2Ctrl": {
            "dataList": null,                       // menu rows (filled in readyFunction)
            "titleConfiguration": "noTitle",
            "numberedList": false,
            "selectCallback": this._menuItemSelectCallback.bind(this)
        }
    },
    "readyFunction": this._readyEntertainment.bind(this),
    "contextInFunction": this._CommCtxtContextIn.bind(this)   // optional
}
```

Field summary: `template`, `templatePath` (app-specific only), `controlProperties`
(per-control config passed to the control constructor), `readyFunction`,
`displayedFunction`, `noLongerDisplayedFunction`, `contextInFunction`,
`contextOutFunction`, `leftBtnStyle` (`goBack`/…), `sbNameId`, `hideHomeBtn`.

## `_messageTable` — native data-message handlers

`{ "<msgId>": function(msg){...} }`. When an MMUI `msg` for this app arrives,
`BaseApp.handleDataMessage` dispatches to `_messageTable[msg.msgId]`. Unknown
msgId → logged warning. This is how native pushes live data into a screen
(e.g. `UpdateEntertainmentMenu`).

## The **menu dataList** schema

A list screen's rows are an array on the control's `dataList`. Each row
(from `systemApp` entertainment list):

```js
{
  appData: {
    appName:     "fmradio",        // target app (or "" / native source)
    isVisible:   true,             // show the row at all
    audioSourceId:"FMRadio",       // for audio sources
    mmuiEvent:   "SelectRadioFM"   // <<-- event sent to MMUI when the row is chosen
  },
  text1Id:  "FmRadio",             // dictionary string id for the label
  disabled: true,                  // greyed out / not selectable
  itemStyle:"style01",
  hasCaret: false                  // chevron (drill-in) indicator
}
```

Literal labels: `List2Ctrl.js` (`/jci/gui/common/controls/List2/js/`) item defaults
include `text1Id: null`, `text1SubMap: null` and `text1: ''` ("textual content of the
label"). The control resolves the dictionary only when `text1Id` is set, and writes
the result into `text1`. A row with `text1Id: null` and `text1: "<text>"` therefore
shows the literal text without a dictionary entry. On the unit, in both
the Applications and the Entertainment list, literal labels render in the stock
font, a row with a `hasCaret` shows the chevron, and an extra Entertainment row
appended to the visible list sits after the last stock row (after AUX) and scrolls
and focuses like a stock row.

Navigation events: on `system.HomeScreen` the icon handler
(`_selectCallbackHomeScreen`) sends `system` `SelectApplications`,
`SelectCommunication`, `SelectEntertainment`, `SelectNavigation` or
`SelectSettings`. `SelectApplications` opens `system.Applications`.
`SelectEntertainment` does not open the Entertainment list: it resumes the active
source (e.g. `fmradio.NowPlaying`). The list `system.Entertainment` opens from the
source's Ent-menu button, which sends `<source uiaId>` `SelectSource`
(`fmradio`: button `IcnUmpEntMenu`). `Global.GoBack` from that NowPlaying returns to
the previous context (HomeScreen), not to the list. `system` `SelectEntertainmentMenu`
is a dictionary id only; as an MMUI event it is ignored.

Entertainment list: `systemApp._readyEntertainment` calls `_buildEntertainmentDataList()`,
which filters `_masterEntertainmentDataList` (built once in the constructor by
`_initEntertainmentDataList`) down to visible items, then passes the result to
`list2Ctrl.setDataList`. That is the same pattern as `_buildApplicationsDataList` for
Applications. `_buildEntertainmentDataList` is also called on `StatusMenuVisible`
updates while the context `Entertainment` is shown.

Selecting the row fires the control's `selectCallback`, which calls
`framework.sendEventToMmui(uiaId, appData.mmuiEvent, ...)`. So **a new menu entry
= a new row object with a `mmuiEvent` native understands + a dictionary id for
its label.** A purely GUI-side action (e.g. open an injected overlay) can
instead have its `selectCallback` call injected code — no native event needed.

Simpler rows (dialogs) use `appData` as a bare event string, e.g.
`appData: "Global.Yes"` / `"Global.GoBack"` / `"SelectConnect"`.

## Templates vs controls

- **Template** = a full screen layout (`<Name>Tmplt`). Instantiated via
  `framework.instantiateTemplate(uiaId, parentDiv, name, properties, path)`.
  Common ones live in `common/templates/<base>/js/<base>Tmplt.js`; app ones need
  `templatePath`. **Common templates:** `ClockSettings2, DialPad2, Dialog3,
  InCall2, Keyboard, List2, Messaging2, NoCtrl, NowPlaying4, ScreenSettings2,
  ScrollDetail, Tuner4`.
- **Control** = a reusable widget inside a template (`<Name>Ctrl`). Instantiated
  via `framework.instantiateControl(...)`; the control appends its own HTML and
  implements `cleanUp()`. **Common controls:** `Button, ClockSettings2, DialPad2,
  Dialog3, InCall2, Keyboard, LeftBtn, List2, Messaging2, NowPlaying4, Sbn,
  ScreenSettings2, ScrollDetail, Slider, StatusBar, Tabs, Tuner4, Ump3, Wink`.

Both resolve the constructor via `window[name]`, so a template/control js must
define a global `function <Name>Tmplt(...)` / `function <Name>Ctrl(...)` and end
with `registerTmpltLoaded`/`registerCtrlLoaded`.

## Localization

- Dictionary file: `<uiaId>AppDict_<lang>.js` — maps string ids (`text1Id`,
  `sbNameId`) to localized text. Centralized apps load from `resources/js/<uiaId>/`;
  others from `apps/<uiaId>/js/`. Which path is decided by
  `guiResources.DICTIONARY_LIST[uiaId]` (see [[05-framework-modules]]).
- 36 languages ship (e.g. `en_US, de_DE, fr_FR, ru_RU, ja_JP, zh_CN, …`).
- Look up at runtime via `framework.localize.getLocStr(defaultAppName, stringId, subMap)`.

## Context category (domain)

`ContextCategory.getContextCategory(uiaId, ctxtId)` maps an app/context to a
**domain**: `Applications, Communication, Entertainment, Settings, Navigation,
Favorites, Other`. The domain drives the status-bar domain icon
(`getContextCategorySbIcon(domain)`). To slot a new app into a domain menu,
its `uiaId.*` needs an entry in that table (see [[apps-inventory]]).
