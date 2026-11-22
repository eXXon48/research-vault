---
tags: [cmu150, vehicle-data, vbs, can, dbus]
---

# Vehicle Data (VBS)

Which vehicle signals reach the CMU150 userland and over which interfaces. Firmware
`cmu150_EU_74.00.324A`, from the stock root filesystem (static analysis only: symbol names,
config files; no live capture). Part of [[CMU150]]; HMI side in [[Custom Apps (HMI)]]; GPS in
[[GPS (LDS)]].

## Data path
```
CAN ─► VIP MCU ─► VIP messages (PIDs, not raw CAN IDs) ─► libjcimod_can (dispatch table)
    ─► VBS function modules (/jci/vbs/modules/libjcimod_*.so) ─► D-Bus com.jci.vbs.*
    ─► native services (vdt, navi, aapa, carplay, blm*, MMUI UIA libs)
    ─► HMI (Opera) receives only what MMUI UIA libs forward
```
- `/jci/vbs/`: `svcjcivbs.so` (SM service `jciVBS`), `vbs.conf`, `vbs-mod-wrapper`.
- `libjcimod_can` groups VIP messages by function (Power Train, ECM Status, Engine Throttle, …).
- `vim_app` also references yaw/brake-control signals.

## Modules and D-Bus names
Modules: `am`, `bcm`, `can`, `cd`, `diag`, `drvididm`, `eem`, `navigation`, `radio`, `settings`,
`vdt`, `vwm`.

D-Bus: `com.jci.vbs.{am,bcm,cd,diag,drvididm,eem,mod_can,navi,navi.tmc,radio,settings,vdt,vwm}`.
Client libraries expose per-signal subscription symbols `VBS_<MOD>_<Signal>_enable`.

## Signals by module (client-library enables)
| Module | Signals (selection) |
|---|---|
| `vdt` | `Veh_V_ActlEng` (vehicle speed), `VehVActlEng_D_Qf` (quality), `EngAout_N_Actl` (engine speed), `ApedPos_Pc_ActlArb` (accelerator), `Lateral_Acceleration`, `VehLong_A_Actl`, `SteWhlComp_An_Est` (steering angle), `OdoCount`, `FuelPulse`, `Ignition_Status`, `GlobalRealTime`, `Total_Distance`, `PCM_DTC`, `TCM_DTC` |
| `bcm` | `CarSpeedResp`, `WheelSpeedNotification`, `GearPosNotification`, `BrakePedalDepressedStatus`, `PrkBrkActvNotification`, `TransmChangeLeverPositionNotification`, `BatteryVoltageFlag`, `NoSpeedRestrict_TouchDisplay`, `VideoRestrictNotification` |
| `vwm` (≈295) | `BoostPressure`, `AirAmb_P_Actl_2`, `AirAmb_Te_Actl_2`, `EngClnt_Te_Actl_2`, `SteeringColumnTorque`, `SteeringAngleChangeRate`, `SteerMtrCurrent_I_Actl`, `BpedMove_D_Actl`, `OilPressureWarning`, tire-pressure status FL/RL/RR, `ShiftIndicatorLight`, `SilGearPos_D_Dsply`, `Turn_Ind_Cmd_Left/Right`, `HeadLght_D_Actl`, `FirstRowBuckleDriver`, `DrvSlipCtlLamp_D_Rq`, `Dsc_B_Falt` |
| `eem` | hybrid / i-stop / eco signals |

**Yaw rate** has no client-library enable. It exists internally: `VBS_VDT_SendSignal_YawRate`
(`libjcimod_vdt.so`), `VBS_BUS_CAN_YawRate_BrakeControl` (`libjcimod_can.so`),
`VDM_UpdateCVD_YawRate` (`libjcivdm.so`), and in `vim_app`.

## Current vehicle data (CVD) database
`/jci/smdb/vdm_vdt_current_data.xml` (≈580 fields) is the VDM/VDT current-data store; per-model
drive-chart PIDs are in `vdm_vdt_pid_data.xml`. Selected fields and stock defaults (defaults
mark "no data"):

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
| `*_Qf` quality flags (brake pressure, long./lat. accel) | 1 |

Also present: i-DM driving-assessment counters (`DR_OpeAsses_*`, `*AssessPoint`), body
customisation states (`CstmzStat_*`), HVAC/lamp/wiper states, hybrid energy flow, battery
analysis records. `smdb_vdm_*.dat` under `/data` is the persisted copy (kept across a
same-flavor update, see [[Hardware and Firmware]]).

## What the HMI web layer gets
- Stock apps `vdt`, `ecoenergy`, `idm` are fed by MMUI UIA libraries
  (`/jci/mmui/uia/libjciuiavdt.so`, `libjciuiaecoenergy.so`, `libjciuiaidm.so`,
  `libjciuiavehsettings.so`).
- They receive derived values only: fuel economy (current / cumulative / average), eco-graph
  data, i-DM assessments, and the speed-restriction state `Global.AtSpeed` / `Global.NoSpeed`.
- No raw speed, RPM, yaw or wheel-speed stream reaches the Opera layer in stock firmware. A
  live gauge needs a native D-Bus client of `com.jci.vbs.*` (or VDT) that forwards values to
  the web layer.

## Open
- D-Bus method/signal signatures and value scaling (client libs ship no introspection XML).
- Update rates per signal.
- Whether wheel speeds are per-wheel in `WheelSpeedNotification`.
