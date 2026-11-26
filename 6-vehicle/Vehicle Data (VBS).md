---
tags: [cmu150, vehicle-data, vbs, can, dbus]
---

# Vehicle Data (VBS)

Which vehicle signals reach the unit's user space, through which modules and D-Bus names, and what
the HMI web layer receives.

## Overview

```
CAN ─> VIP MCU ─> VIP messages (PIDs, not raw CAN IDs) ─> libjcimod_can (dispatch table)
    ─> VBS function modules (/jci/vbs/modules/libjcimod_*.so) ─> D-Bus com.jci.vbs.*
    ─> native services (vdt, navi, aapa, carplay, blm*, MMUI modules)
    ─> HMI (Opera): only what the MMUI modules forward
```

Speed, RPM, steering, boost and many other signals are available on D-Bus. The Opera layer receives
only derived values (fuel economy, i-DM assessments, the at-speed state). A live gauge needs a
native D-Bus client of `com.jci.vbs.*` that forwards values to the web layer.

Related: [[Custom Apps (HMI)]], [[GPS (LDS)]], [[Vehicle VIN]], [[Touch Speed Restriction]],
[[README|CMU150 research]].

## Reference

### Components

- `/jci/vbs/`: `svcjcivbs.so` (SM service `jciVBS`), `vbs.conf`, `vbs-mod-wrapper`.
- `libjcimod_can` groups VIP messages by function (Power Train, ECM Status, Engine Throttle, …).
- `vim_app` also references yaw and brake-control signals.

### Modules and D-Bus names

- Modules: `am`, `bcm`, `can`, `cd`, `diag`, `drvididm`, `eem`, `navigation`, `radio`, `settings`,
  `vdt`, `vwm`.
- D-Bus names: `com.jci.vbs.{am,bcm,cd,diag,drvididm,eem,mod_can,navi,navi.tmc,radio,settings,vdt,vwm}`.
- Client libraries expose one subscription symbol per signal: `VBS_<MOD>_<Signal>_enable`.

### Signals by module

From the client-library enables (selection):

| Module | Signals |
|---|---|
| `vdt` | `Veh_V_ActlEng` (vehicle speed), `VehVActlEng_D_Qf` (quality), `EngAout_N_Actl` (engine speed), `ApedPos_Pc_ActlArb` (accelerator), `Lateral_Acceleration`, `VehLong_A_Actl`, `SteWhlComp_An_Est` (steering angle), `OdoCount`, `FuelPulse`, `Ignition_Status`, `GlobalRealTime`, `Total_Distance`, `PCM_DTC`, `TCM_DTC` |
| `bcm` | `CarSpeedResp`, `WheelSpeedNotification`, `GearPosNotification`, `BrakePedalDepressedStatus`, `PrkBrkActvNotification`, `TransmChangeLeverPositionNotification`, `BatteryVoltageFlag`, `NoSpeedRestrict_TouchDisplay`, `VideoRestrictNotification` |
| `vwm` (≈295 signals) | `BoostPressure`, `AirAmb_P_Actl_2`, `AirAmb_Te_Actl_2`, `EngClnt_Te_Actl_2`, `SteeringColumnTorque`, `SteeringAngleChangeRate`, `SteerMtrCurrent_I_Actl`, `BpedMove_D_Actl`, `OilPressureWarning`, tire-pressure status FL/RL/RR, `ShiftIndicatorLight`, `SilGearPos_D_Dsply`, `Turn_Ind_Cmd_Left/Right`, `HeadLght_D_Actl`, `FirstRowBuckleDriver`, `DrvSlipCtlLamp_D_Rq`, `Dsc_B_Falt` |
| `eem` | hybrid, i-stop and eco signals |

Yaw rate has no client-library enable. It exists internally: `VBS_VDT_SendSignal_YawRate`
(`libjcimod_vdt.so`), `VBS_BUS_CAN_YawRate_BrakeControl` (`libjcimod_can.so`),
`VDM_UpdateCVD_YawRate` (`libjcivdm.so`), and in `vim_app`.

### Current vehicle data (CVD) database

`/jci/smdb/vdm_vdt_current_data.xml` (≈580 fields) is the VDM/VDT current-data store. Per-model
drive-chart PIDs are in `vdm_vdt_pid_data.xml`. Selected fields and stock defaults (a default value
means "no data"):

| Field | Default |
|---|---|
| `VehicleSpeed`, `EngineSpeed`, `AccelPedalPosition` | 0 |
| `SteeringWheel` | 32766 |
| `LongitudinalAccelerometer`, `LateralAcceleration` | 4000 |
| `YawRate` | 4094 |
| `BrakePressure` | 152 |
| `LatitudeByGPS`, `LongitudeByGPS`, `FuelPulse`, `Odocount` | 0 |
| `TransmissionGearPosition`, `TransmChangeLeverPosition`, `EngineLoad`, `EngClnt_Te_Actl`, `DR_IntakeAirTemp`, `DR_Slope`, `DR_ClutchSW`, `DR_NeutralSW` | 0 |
| `Battery_StateOfCharge` | 254 |
| `*_Qf` quality flags (brake pressure, longitudinal and lateral acceleration) | 1 |

- Also present: i-DM driving-assessment counters (`DR_OpeAsses_*`, `*AssessPoint`), body
  customisation states (`CstmzStat_*`), HVAC, lamp and wiper states, hybrid energy flow, battery
  analysis records.
- `smdb_vdm_*.dat` under `/data` is the persisted copy. It is kept across a same-flavor update
  ([[Firmware Update]]).

### What the HMI web layer receives

- The stock apps `vdt`, `ecoenergy` and `idm` are fed by MMUI modules
  (`/jci/mmui/uia/libjciuiavdt.so`, `libjciuiaecoenergy.so`, `libjciuiaidm.so`,
  `libjciuiavehsettings.so`).
- They receive derived values only: fuel economy (current, cumulative, average), eco-graph data,
  i-DM assessments, and the speed-restriction state `Global.AtSpeed` / `Global.NoSpeed`.
- No raw speed, RPM, yaw or wheel-speed stream reaches the Opera layer in stock firmware.

## Verified on

Stock rootfs of firmware `cmu150_EU_74.00.324A` (static analysis: symbol names and configuration
files; no live capture).

## Open questions

- D-Bus method and signal signatures and value scaling (the client libraries ship no introspection
  XML).
- Update rate per signal.
- Whether `WheelSpeedNotification` carries per-wheel speeds.
