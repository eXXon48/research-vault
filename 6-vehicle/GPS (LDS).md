---
tags: [cmu150, gps, lds, vehicle-data]
---

# GPS (LDS)

The Location Data Service (LDS), which owns the GPS receiver: its files, configuration and how the
receiver type is chosen.

## Overview

LDS is the SM service `jciLDS` (`/jci/lds/svcjcilds.so`). It reads NMEA from the receiver on
`/dev/ttymxc2` and publishes data on D-Bus (`com.jci.lds.control`, `com.jci.lds.data`). The legacy
board has an ST Teseo-family receiver; the WCP board has a u-blox receiver.

Related: [[Hardware and Firmware]] ("Board variants and SM configuration"),
[[Vehicle Data (VBS)]], [[Custom Apps (HMI)]], [[README|CMU150 research]].

## Reference

### Files (`/jci/lds/`)

| File | Role |
|---|---|
| `svcjcilds.so` | LDS service (SM service `jciLDS`) |
| `lds.xml` | configuration |
| `libjcilds-ublox.so` | u-blox support (referenced by `svcjcilds.so`) |
| `ubx_nmea` | ARM ELF helper (14 KB) |
| `lds-dbus-control-tester`, `lds-dbus-data-tester`, `lds-dbus-nmea-tester` | stock D-Bus test clients |
| `instConf.xml` | — |

### `lds.xml` (stock values)

| Setting | Value |
|---|---|
| `LDS_GPSDataSerialPort0` | `/dev/ttymxc2` (primary port; no secondary port) |
| `LDS_StartupMode` | `1`: LDS waits for D-Bus start/stop signals before processing GPS data |
| `LDS_InvalidDataTimeout` | 5 s |
| `LDS_NMEALoggerMode` | `0` (off); the log path would be `/jci/lds/nmea.log` |
| GPS hardware control | on for platforms `cmu140,cmu150,volans` |
| Firmware query | `$PSTMGETPAR,1500` |
| Ignore list | many `$PSTM*` sentences |
| Antenna supervision (ADC) | voltage channel 0 (660–734), current channel 4 (157–544) |

`$PSTM…` is the proprietary NMEA family of ST Teseo (STA80xx) receivers.

### Receiver selection

- `svcjcilds.so` has both paths: `LDS_UBX_isUblox`, `LDS_UBX_isJP`,
  `LDS_UBX_GPS_PowerUp/Down/Reset`, `LDS_DBUS_Ublox_PowerOff`, baud-rate setup.
- `sm_WCP.conf` passes `GPS=UBLOX` to `jciLDS` and adds a `stage_navi` dependency. `sm.conf` (legacy
  board) passes no GPS argument.

| Board | Receiver | NMEA |
|---|---|---|
| legacy | ST Teseo family | `$PSTM` proprietary sentences |
| WCP | u-blox | UBX |

## Verified on

Stock rootfs of firmware `cmu150_EU_74.00.324A`.

## Open questions

- The exact receiver parts and their maximum fix rates. Public sources name the u-blox part NEO-M8L
  and the older receiver STA8088, and report about 2 fixes/s on STA8088 and about 10/s on u-blox,
  against 1 Hz in stock. None of this is confirmed on the unit.
