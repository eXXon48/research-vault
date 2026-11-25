---
tags: [cmu150, vin, dbus, vehicle]
---

# CMU150 Vehicle VIN

How the vehicle VIN can be read on the CMU150. Firmware `74.00.324A`.
Related: [[CMU150 Users and Privileges]].

## D-Bus method
- Service `com.jci.vbs.navi`, object `/com/jci/vbs/navi`, interface `com.jci.vbs.navi`.
- Method `GetVINData`: no input; output `vinData`, type `s`. Introspection marks it
  `async` client-side.
- Bus: the JCI service socket `unix:path=/tmp/dbus_service_socket` (the same bus as
  `com.jci.blm.system`).
- The introspection XML is embedded in `/jci/vbs/modules/libjcimod_navigation.so`. The
  client stub is `VBS_NAVI_GetVINData` in `/jci/lib/libjcivbsnaviclient.so`.
- Callers in the stock image include `svcjcinavi.so`, `svcjciblm_ira_aha.so`,
  `svcjciusbdtc.so`, `svcjciblmsettings.so`, `svcjcitestdiag.so` and `/jci/dataretrieval/dataRetrieval`.

Call from a shell:

```sh
/usr/bin/dbus-send --address=unix:path=/tmp/dbus_service_socket --print-reply \
  --reply-timeout=5000 --type=method_call --dest=com.jci.vbs.navi \
  /com/jci/vbs/navi com.jci.vbs.navi.GetVINData
```

## Related signal
- `GccConfigMgmtResp` on the same interface carries `vin_Character`, type `(ay)` (struct
  `VBS_NAVI_GCC_ConfigMgmt_Mde_VIN_Data_s_t`). This is the per-byte form of the VIN.
- `/jci/lib/libjcivdm.so` logs `VIN_character %d = <%c>`.

## Behaviour
- With no vehicle attached, the call returns a 17-character VIN string (`<VIN>`). The
  VIN is stored on the unit and does not need live CAN traffic at call time.
- A stored VIN observed on a unit starts with `JMZND`.
- The call works as uid 0. The system-bus policy allows method calls to root only; whether
  this service socket enforces the same policy is not established.

## Model from the VIN
Public reference tables (Wikibooks "Vehicle Identification Numbers (VIN codes)/Mazda/VIN Codes",
Wikipedia "List of Mazda model codes"), not firmware.

- Positions 1-3 (WMI), Mazda:
  - `JM1` passenger car, `JM3` MPV/SUV (Japan-built, North-America layout).
  - `JMZ` Europe and Israel export; `JM0` Oceania export; `JM4` `JM6` `JM7` other export.
  - `3MD` Mazda2, `3MZ` Mazda3, `3MJ` CX-3 (Mexico); `JC1` Mazda-built Fiat 124 Spider.
- Positions 4-5 are the Mazda model (chassis) code on Mazda-style VINs. Codes of the models sold
  with this head unit:

  | Code | Model |
  |---|---|
  | `DJ` | Mazda2 / Demio (hatch) |
  | `DL` | Mazda2 sedan; on Mexico-built `3MY` VINs, the Toyota-badged Scion iA / Yaris |
  | `BM`, `BN` | Mazda3 / Axela (BN = BM facelift, not Japan) |
  | `BY` | Axela Hybrid (Japan only) |
  | `GJ`, `GL` | Mazda6 / Atenza (GL = GJ refresh) |
  | `DK` | CX-3 |
  | `GK` | CX-4 |
  | `KE`, `KF` | CX-5 gen 1, gen 2 |
  | `KG` | CX-8 |
  | `TC` | CX-9 gen 2 |
  | `ND` | MX-5 gen 4 |
  | `NF` | Fiat 124 Spider |

  Later codes (`BP` Mazda3, `DM` CX-30, `DR` MX-30, `KH`/`KK`/`KL` CX-60/90/80, `VA` CX-50) belong to
  the next head-unit generation.
- The prefix `JMZND` decodes as an MX-5 (ND), Europe export. The unit's `/data_persist/model.txt`
  reads `Mazda_SB2` (the head unit, not the car).
- North-America layout only: position 6 restraint, 7 body, 8 engine (per model: the same letter
  means different engines on different models), 9 check digit, 10 model year, 11 plant
  (`0` Hiroshima, `1` Hofu, `M` Salamanca). Export layouts (`JMZ`, `JM0`, `JM7`) use positions 6-11
  as one series descriptor: no reliable year, engine or trim.

## Model from the HMI (no VIN needed)
- The HMI keeps a vehicle program code: `framework.getSharedData('syssettings','VehicleType')`.
  It is set by `syssettingsApp` from the native `VehicleType` message (`payload.vtype`).
  `GuiFramework.js` defaults it to `J36` when the message is missing.
- Codes referenced in the stock GUI, as `SETTINGS_VehicleModelType_*`:
  - `J03A` `J03AE` `J03E` `J03F` `J03G` `J03J` `J03K` `J03W`
  - `J12` `J12A` `J12AE` `J12F`
  - `J36` `J36IPM`
  - `J53` `J71` `J72A` `J78A`
- The mapping from J-code to model is not in the firmware strings.

## Colour
- Mazda VINs do not encode the paint colour. The paint code is printed on the body
  identification plate.
- No exterior-colour field exists in the stock GUI (`/jci/gui`). The unit has no known
  source for the body colour.
