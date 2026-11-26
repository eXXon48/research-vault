---
tags: [cmu150, gps, lds, vehicle-data]
---

# GPS (LDS)

GPS on the CMU150 is owned by the Location Data Service (LDS). Firmware `cmu150_EU_74.00.324A`.
Part of [[README|CMU150 research]]; board/config selection in [[Hardware and Firmware]] ("Board type and SM
config selection"); vehicle-bus data path in [[Custom Apps (HMI)]].

## Files (`/jci/lds/`)
| File | Role |
|---|---|
| `svcjcilds.so` | LDS service (SM service `jciLDS`) |
| `lds.xml` | configuration (below) |
| `libjcilds-ublox.so` | u-blox support (referenced by `svcjcilds.so`) |
| `ubx_nmea` | ARM ELF helper (14 KB) |
| `lds-dbus-control-tester`, `lds-dbus-data-tester`, `lds-dbus-nmea-tester` | stock D-Bus test clients |
| `instConf.xml` | — |

D-Bus interfaces: `com.jci.lds.control`, `com.jci.lds.data`.

## `lds.xml` (stock values)
- Primary GPS serial port `LDS_GPSDataSerialPort0 = /dev/ttymxc2`; no secondary port.
- `LDS_StartupMode = 1`: LDS waits for D-Bus start/stop signals before processing GPS data.
- `LDS_InvalidDataTimeout = 5` s.
- NMEA logger off (`LDS_NMEALoggerMode = 0`); log path would be `/jci/lds/nmea.log`.
- GPS hardware control on for platforms `cmu140,cmu150,volans`.
- Firmware query string `$PSTMGETPAR,1500`; the ignore list holds many `$PSTM*` sentences.
  `$PSTM…` is the ST Teseo (STA80xx) proprietary NMEA family.
- Antenna supervision via ADC: voltage channel 0 (660–734), current channel 4 (157–544).

## Receiver selection
- `svcjcilds.so` has both paths: `LDS_UBX_isUblox`, `LDS_UBX_isJP`, `LDS_UBX_GPS_PowerUp/Down/Reset`,
  `LDS_DBUS_Ublox_PowerOff`, baud-rate setup.
- `sm_WCP.conf` passes `GPS=UBLOX` to `jciLDS` and adds a `stage_navi` dependency; `sm.conf`
  (legacy board) passes no GPS argument. Legacy board → ST Teseo-family receiver with `$PSTM`
  NMEA; WCP board → u-blox.
- Community sources name the u-blox part NEO-M8L and the older receiver STA8088, and report
  ~2 fixes/s achievable on STA8088 vs ~10/s on u-blox, 1 Hz stock. Not verified here.
