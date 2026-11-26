---
tags: [cmu150, vin, dbus, vehicle]
---

# Vehicle VIN

How to read the vehicle VIN on the unit, how to decode the model from it, and what the HMI knows
about the vehicle without the VIN.

## Overview

The VIN is stored on the unit and returned by the D-Bus method `GetVINData` of `com.jci.vbs.navi`,
with no live CAN traffic needed. Positions 4–5 of a Mazda-style VIN give the model code. The HMI
separately keeps a vehicle program code (`J36` and others). Neither source gives the paint colour.

Related: [[Vehicle Data (VBS)]], [[Users and Privileges]].

## Reference

### D-Bus method

| Item | Value |
|---|---|
| Service | `com.jci.vbs.navi` |
| Object | `/com/jci/vbs/navi` |
| Interface | `com.jci.vbs.navi` |
| Method | `GetVINData`: no input; output `vinData`, type `s`. Introspection marks it `async` on the client side. |
| Bus | the JCI service bus, `unix:path=/tmp/dbus_service_socket` (the same bus as `com.jci.blm.system`) |

- The introspection XML is embedded in `/jci/vbs/modules/libjcimod_navigation.so`. The client stub is
  `VBS_NAVI_GetVINData` in `/jci/lib/libjcivbsnaviclient.so`.
- Callers in the stock image include `svcjcinavi.so`, `svcjciblm_ira_aha.so`, `svcjciusbdtc.so`,
  `svcjciblmsettings.so`, `svcjcitestdiag.so` and `/jci/dataretrieval/dataRetrieval`.
- With no vehicle attached, the call returns a 17-character VIN string. The VIN is stored on the
  unit.

### Related signal

- `GccConfigMgmtResp` on the same interface carries `vin_Character`, type `(ay)` (struct
  `VBS_NAVI_GCC_ConfigMgmt_Mde_VIN_Data_s_t`): the VIN byte by byte.
- `/jci/lib/libjcivdm.so` logs `VIN_character %d = <%c>`.

### Model from the VIN

From public reference tables (Wikibooks "Vehicle Identification Numbers (VIN codes)/Mazda/VIN
Codes", Wikipedia "List of Mazda model codes"), not from the firmware.

**Positions 1–3 (WMI), Mazda:**

| WMI | Meaning |
|---|---|
| `JM1` | passenger car (Japan-built, North-America layout) |
| `JM3` | MPV / SUV (Japan-built, North-America layout) |
| `JMZ` | Europe and Israel export |
| `JM0` | Oceania export |
| `JM4`, `JM6`, `JM7` | other export |
| `3MD`, `3MZ`, `3MJ` | Mazda2, Mazda3, CX-3 (Mexico) |
| `JC1` | Mazda-built Fiat 124 Spider |

**Positions 4–5: model (chassis) code** on Mazda-style VINs. Models sold with this head unit:

| Code | Model |
|---|---|
| `DJ` | Mazda2 / Demio (hatch) |
| `DL` | Mazda2 sedan; on Mexico-built `3MY` VINs, the Toyota-badged Scion iA / Yaris |
| `BM`, `BN` | Mazda3 / Axela (BN = BM facelift, not Japan) |
| `BY` | Axela Hybrid (Japan only) |
| `GJ`, `GL` | Mazda6 / Atenza (GL = GJ refresh) |
| `DK` | CX-3 |
| `GK` | CX-4 |
| `KE`, `KF` | CX-5 generation 1, generation 2 |
| `KG` | CX-8 |
| `TC` | CX-9 generation 2 |
| `ND` | MX-5 generation 4 |
| `NF` | Fiat 124 Spider |

Later codes (`BP` Mazda3, `DM` CX-30, `DR` MX-30, `KH`/`KK`/`KL` CX-60/90/80, `VA` CX-50) belong to
the next head-unit generation.

**Positions 6–11.** North-America layout only: 6 restraint, 7 body, 8 engine (per model; the same
letter means different engines on different models), 9 check digit, 10 model year, 11 plant (`0`
Hiroshima, `1` Hofu, `M` Salamanca). Export layouts (`JMZ`, `JM0`, `JM7`) use positions 6–11 as one
series descriptor, with no reliable year, engine or trim.

`/data_persist/model.txt` reads `Mazda_SB2`: the head unit model, not the car.

### Model from the HMI

- The HMI keeps a vehicle program code: `framework.getSharedData('syssettings','VehicleType')`.
  `syssettingsApp` sets it from the native `VehicleType` message (`payload.vtype`). `GuiFramework.js`
  defaults it to `J36` when the message is missing.
- Codes referenced in the stock GUI as `SETTINGS_VehicleModelType_*`:
  - `J03A` `J03AE` `J03E` `J03F` `J03G` `J03J` `J03K` `J03W`
  - `J12` `J12A` `J12AE` `J12F`
  - `J36` `J36IPM`
  - `J53` `J71` `J72A` `J78A`
- The mapping from J-code to model is not in the firmware strings.

### Colour

- Mazda VINs do not encode the paint colour. The paint code is printed on the body identification
  plate.
- The stock GUI (`/jci/gui`) has no exterior-colour field. The unit has no known source for the body
  colour.

## How to: read the VIN from a shell

```sh
/usr/bin/dbus-send --address=unix:path=/tmp/dbus_service_socket --print-reply \
  --reply-timeout=5000 --type=method_call --dest=com.jci.vbs.navi \
  /com/jci/vbs/navi com.jci.vbs.navi.GetVINData
```

The call works as uid 0.

## Verified on

Firmware `cmu150_EU_74.00.324A`: stock rootfs for the interface, bench unit (no vehicle attached)
for the call.

## Open questions

- Whether a non-root process can make the call (see [[Users and Privileges]], "D-Bus").
