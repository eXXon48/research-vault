# Touch Speed Restriction

Firmware 74.00.324A blocks touch input while the car moves. Two independent layers do it.
Both must be off for touch to work at speed.

## Layer 1: HMI (Opera JS)

`/jci/gui/common/js/Common.js` (rootfs, read-only) registers the MMUI speed messages:

```js
        "Global.AtSpeed" : this._AtSpeedMsgHandler.bind(this),
```

(line 152 in the stock file, 8-space indent). `_AtSpeedMsgHandler` (line ~715) sets
`this._atSpeed = true` and forwards the message to the focused app, which then locks its touch
controls. `_NoSpeedMsgHandler` clears it; `getAtSpeedValue()` reads it.

Commenting out the `"Global.AtSpeed"` line stops the HMI from entering the at-speed state.
Known community mods (MZD-AIO tweak `01_touchscreen-i`, TouchTune) do exactly this one-line edit.

## Layer 2: NVRAM keys (native)

Two keys in the `nv-config` NVRAM, values `enable` | `disable`:

| Key | Stock setter | Native readers |
|-----|--------------|----------------|
| `bus_bcm_speed_restriction` | `/jci/scripts/set_speed_restriction_config.sh enable\|disable` | `/jci/vbs/modules/libjcimod_bcm.so` |
| `lvds_speed_restriction` | `/jci/scripts/set_lvds_speed_restriction_config.sh enable\|disable` | `/jci/lvds/lvds_service`, `/jci/nng/jci-linux_imx6_volans-release` |

Each setter does three writes:

```sh
echo <key>=<value> > /sys/class/nvram/nv-config/add
echo <value>       > /sys/class/nvram/nv-config/keys/<key>
echo ""            > /sys/class/nvram/nv-config/commit
```

- Current value: `cat /sys/class/nvram/nv-config/keys/<key>`.
- A key can be absent until a setter first runs. Absent = factory state.
- The value survives reboots (NVRAM, not a file). It is not on rootfs or `/data`, so a file-based
  restore does not reset it. A reflash `--restore` with a flavor mismatch erases the `nv-config` MTD.
- Stock diag menu toggles both: `/jci/scripts/systemHooks.sh` cases
  `SYSTEM_SPEED_RESTRICTION_TOGGLE=4` (bus_bcm) and `LVDS_SPEED_RESTRICTION_TOGGLE=31` (lvds).

## Other nv-config keys (stock rootfs references)

Key names found as strings in stock scripts and binaries. The full live set is
`ls /sys/class/nvram/nv-config/keys` on the unit (`dataRetrieval` dumps all of them the same way).
Native services read keys through `/jci/lib/libjcinvram.so` (`NVRAM_Block_get`); the kernel side is
`com_jci_cpp_nvram.ko`.

| Key | Values (where known) | Read / set by |
|-----|----------------------|---------------|
| `bus_bcm_speed_restriction` | `enable` \| `disable` | libjcimod_bcm.so; setter script; diag toggle 4 |
| `lvds_speed_restriction` | `enable` \| `disable` | lvds_service; setter script; diag toggle 31 |
| `lvds_hdcp_mode` | `enable` \| `disable` | lvds_service; `set_hdcp_config.sh`; diag toggle 1 |
| `lvds_display_type` | `visteon` (default when absent) \| `jci` | lvds_service; `set_display_config.sh`; diag toggle 13 |
| `lvds_bright_mode`, `lvds_bright_day`, `lvds_bright_night` | not determined | lvds_service, svcjciblmsettings.so, svcjcivideoctrl.so |
| `lvds_contrast_day`, `lvds_contrast_night` | not determined | lvds_service, svcjciblmsettings.so |
| `lvds_camera_detection` | not determined | lvds_service, svcjcivideoctrl.so |
| `backup_camera_type` | not determined | svcjciblmsettings.so, svcjcinativegui(.so/ctrl) |
| `region_info` | not determined | lvds_service, svcjciblmsettings.so |
| `tuneLvds_boost`, `tuneLvds_eqtune` | not determined | lvds_service (LVDS link signal tuning) |

`lvds_display_type`, `tuneLvds_*` and `region_info` describe the hardware/market of the unit; a
wrong value can leave the display dark or change regional behaviour. Brightness/contrast keys are
written by the settings service from the user's display settings.

Source: stock rootfs image, firmware 74.00.324A.
