---
tags: [cmu150, hmi, touch, nvram]
---

# Touch Speed Restriction

How the unit blocks touch input while the car moves, and the `nv-config` NVRAM keys involved.

## Overview

Two independent layers block touch at speed. Both must be off for touch to work while moving:

1. **HMI (Opera JS):** the `Global.AtSpeed` message puts the GUI into an at-speed state, and the
   focused app locks its touch controls.
2. **NVRAM keys (native):** `bus_bcm_speed_restriction` and `lvds_speed_restriction` in the
   `nv-config` NVRAM.

Related: [[4-hmi/jci-ui/README|HMI framework reference]], [[Vehicle Data (VBS)]],
[[Partitions and Storage]].

## Reference

### HMI layer

`/jci/gui/common/js/Common.js` (rootfs, read-only) registers the MMUI speed messages. Line 152 of
the stock file, 8-space indent:

```js
        "Global.AtSpeed" : this._AtSpeedMsgHandler.bind(this),
```

| Function | Effect |
|---|---|
| `_AtSpeedMsgHandler` (line ~715) | sets `this._atSpeed = true` and forwards the message to the focused app, which locks its touch controls |
| `_NoSpeedMsgHandler` | clears the state |
| `getAtSpeedValue()` | reads the state |

With the `"Global.AtSpeed"` line commented out, the HMI never enters the at-speed state.

### NVRAM layer

Two keys in the `nv-config` NVRAM, values `enable` | `disable`:

| Key | Stock setter | Native readers |
|---|---|---|
| `bus_bcm_speed_restriction` | `/jci/scripts/set_speed_restriction_config.sh enable\|disable` | `/jci/vbs/modules/libjcimod_bcm.so` |
| `lvds_speed_restriction` | `/jci/scripts/set_lvds_speed_restriction_config.sh enable\|disable` | `/jci/lvds/lvds_service`, `/jci/nng/jci-linux_imx6_volans-release` |

- A key can be absent until a setter first runs. Absent = factory state.
- The value survives reboots (NVRAM, not a file). It is not on the rootfs or `/data`, so a
  file-based restore does not reset it. A reflash `--restore` with a flavor mismatch erases the
  `nv-config` MTD.
- The stock diagnostics menu toggles both keys: `/jci/scripts/systemHooks.sh` cases
  `SYSTEM_SPEED_RESTRICTION_TOGGLE=4` (`bus_bcm`) and `LVDS_SPEED_RESTRICTION_TOGGLE=31` (`lvds`).

### Other `nv-config` keys

Key names found as strings in stock scripts and binaries. Native services read keys through
`/jci/lib/libjcinvram.so` (`NVRAM_Block_get`); the kernel side is `com_jci_cpp_nvram.ko`.

| Key | Values (where known) | Read / set by |
|---|---|---|
| `bus_bcm_speed_restriction` | `enable` \| `disable` | `libjcimod_bcm.so`; setter script; diag toggle 4 |
| `lvds_speed_restriction` | `enable` \| `disable` | `lvds_service`; setter script; diag toggle 31 |
| `lvds_hdcp_mode` | `enable` \| `disable` | `lvds_service`; `set_hdcp_config.sh`; diag toggle 1 |
| `lvds_display_type` | `visteon` (default when absent) \| `jci` | `lvds_service`; `set_display_config.sh`; diag toggle 13 |
| `lvds_bright_mode`, `lvds_bright_day`, `lvds_bright_night` | not determined | `lvds_service`, `svcjciblmsettings.so`, `svcjcivideoctrl.so` |
| `lvds_contrast_day`, `lvds_contrast_night` | not determined | `lvds_service`, `svcjciblmsettings.so` |
| `lvds_camera_detection` | not determined | `lvds_service`, `svcjcivideoctrl.so` |
| `backup_camera_type` | not determined | `svcjciblmsettings.so`, `svcjcinativegui` (`.so` / ctrl) |
| `region_info` | not determined | `lvds_service`, `svcjciblmsettings.so` |
| `tuneLvds_boost`, `tuneLvds_eqtune` | not determined | `lvds_service` (LVDS link signal tuning) |

- `lvds_display_type`, `tuneLvds_*` and `region_info` describe the hardware and market of the unit.
  A wrong value can leave the display dark or change regional behaviour.
- The settings service writes the brightness and contrast keys from the user's display settings.

## How to

**Read a key:**

```sh
cat /sys/class/nvram/nv-config/keys/<key>
```

**List all keys** (the stock `dataRetrieval` dumps them the same way):

```sh
ls /sys/class/nvram/nv-config/keys
```

**Set a key.** Each stock setter does three writes:

```sh
echo <key>=<value> > /sys/class/nvram/nv-config/add
echo <value>       > /sys/class/nvram/nv-config/keys/<key>
echo ""            > /sys/class/nvram/nv-config/commit
```

## Verified on

Stock rootfs image, firmware `cmu150_EU_74.00.324A`.
