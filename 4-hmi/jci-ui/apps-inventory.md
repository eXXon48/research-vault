---
tags: [cmu150, hmi, jci-ui, reference]
---

# Apps inventory

42 app directories under `/jci/gui/apps/`. Each is a `uiaId`; path
`apps/<uiaId>/js/<uiaId>App.js` (see [[04-app-anatomy]]). **Domain** is from
`ContextCategory` (`<uiaId>.*` rule; see [[05-framework-modules]]) — it drives the
status-bar domain icon and which top-level menu the app belongs to. "—" = no
`<uiaId>.*` entry, so it falls through to `Other` unless a specific `uiaId.ctxtId`
is listed.

| uiaId | Domain | Notes |
|-------|--------|-------|
| aharadio | Entertainment | Aha internet radio |
| amradio | Entertainment | |
| androidauto | — (Other) | Android Auto projection |
| audiosettings | Settings | |
| auxin | Entertainment | Aux input |
| backupparking | Other (`backupparking.*`) | Backup/parking |
| btaudio | Entertainment | Bluetooth audio |
| btpairing | Settings | BT pairing |
| carplay | Applications / Settings | per-ctxt: DeviceList/Device→Applications, CPDevice→Settings |
| cd | Entertainment | |
| contacts | Communication | |
| dab | Entertainment | DAB radio (EU) |
| diag | Other | Diagnostics (Opera, 0-byte css) |
| driverid | Applications | |
| dvd | Entertainment | |
| ecoenergy | Applications | |
| email | Communication | |
| emnavi | Navigation | Embedded nav |
| favorites | Favorites | |
| fmradio | Entertainment | TADialog ctxts → Other |
| hdtrafficimage | Applications | region-specific |
| idm | Applications | |
| mobile911 | Communication | |
| netmgmt | Settings | Network mgmt |
| pandora | Entertainment | |
| phone | Communication | |
| satradio | Entertainment | |
| schedmaint | Applications | Scheduled maintenance |
| sms | Communication | |
| stitcher | Entertainment | |
| syssettings | Settings | several ctxts → Communication; drives boot startup settings |
| system | Other / per-ctxt | HomeScreen + domain menus; `system.Applications`→Applications, `.Communication`→Communication, `.Entertainment`→Entertainment, etc. |
| sysupdate | Settings | OTA/update UI |
| tutorial | Other | |
| tv | Entertainment | |
| usbaudio | Entertainment | |
| vdt | Applications | Vehicle data? |
| vehsettings | Settings | Vehicle settings |
| vsm | Applications | Vehicle Status Monitor |
| warnguide | Applications | Warning guide |
| xmaudio | Entertainment | SiriusXM audio |
| xmdata | Applications | SiriusXM data services |

Not a dir but present in manifests: `screenrep` (dicts off), `siri` (dict on).

**Centralized dictionaries** (`guiResources.DICTIONARY_LIST` truthy → dicts under
`resources/js/<uiaId>/`): all of the above except `screenrep` (false). Apps not
in the list fall back to `apps/<uiaId>/js/` for dicts when `hasLangSupport`.

**`system` is the shell.** It owns `HomeScreen` (MainMenu) and the top-level
domain menus (Entertainment/Applications/Communication/VehicleStatusMonitor),
built from data lists in `systemApp.js`. The lists live on the rootfs; scripts loaded
through the `/data` injection hook can patch them at runtime.

## Wi-Fi settings path (user-facing)

Home → **Settings** → **Devices** tab → **Wi-Fi** opens the `netmgmt` app (Wi-Fi on/off, network
list, "Enter Password", "Saved Networks"). Sources (stock rootfs):
- `apps/syssettings/js/syssettingsApp.js`: `_DevicesTabCtxtDataList` = Bluetooth, Apple CarPlay,
  Android Auto, `SelectNetworkManagement`; the tab's `labelId` is `common.DevicesTab`.
- `resources/js/syssettings/syssettingsAppDict_en_US.js`: `"NetworkManagement" : "Wi-Fi"`;
  common dict: `"DevicesTab" : "Devices"`.
- `resources/js/netmgmt/netmgmtAppDict_en_US.js`: `"NetworkConnection" : "Wi-Fi"`, `"EnterPassword"`.
