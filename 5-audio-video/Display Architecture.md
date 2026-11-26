---
tags: [cmu150, display, wayland]
---

# Display Architecture

The display stack: the Wayland compositor and IVI layers, the three rendering clients, and which
screens work without Opera.

## Overview

The display is Wayland, with the compositor `wayland-viv-compositor` (Vivante GPU) and GENIVI
IVI-shell layer management (`wl_ivi_shell`, `libwayland-ivi-*`). Clients draw independent surfaces
onto z-ordered IVI layers. Three clients render; only one of them is Opera.

A native IVI-shell client has its own layer above the HMI layer, so it keeps drawing when Opera
shows a black screen. It needs the compositor and VIM, so it can run only after SM has started
them.

Related: [[Audio Architecture]], [[Screen Capture]], [[01-architecture-and-boot]],
[[Boot Timeline]] ("Screens during boot").

## Reference

### Rendering clients

| Client | Kind | What it draws |
|---|---|---|
| `/jci/nativegui/svcjcinativeguictrl` (+ `svcjcinativegui.so`) | native Wayland | static PNGs only: boot splash, SafetyText (warning not to watch while driving), NoVideoSignal, PSMOpen (~101 images) |
| `/jci/cpugauge/jci_cpu_gauge` | native Wayland + Cairo | live text overlay (CPU, memory, threads, Linux version) |
| Opera / `jciMMUI` | Opera | the HMI: the JS/HTML apps under `/jci/gui` |

### `jci_cpu_gauge`

A standalone native Wayland client, 76 475 bytes, ARMv7 ELF. From `DT_NEEDED` and the dynamic
symbols:

| Item | Content |
|---|---|
| `NEEDED` | `libwayland-client.so.0`, `libwayland-ivi-client.so.0`, `libwayland-ivi-shell-client.so.1`, `libcairo.so.2`, `libjcivim_api.so`, plus `libjcicommon`, `libjcipc`, `libjcipcapi`, `libstdc++`, `libm`, `libpthread`, `libc` |
| Wayland | `wl_display_connect`, `wl_proxy_marshal*`; binds `wl_compositor`, `wl_shell`, `wl_ivi_shell`, `wl_shm`; `wl_shm_create_pool` / `wl_shm_create_buffer` (its own buffer) |
| Drawing | `cairo_*`: `cairo_image_surface_create_for_data`, `cairo_show_text`, move, rect, stroke |
| Strings | `ws_tm_draw`, `draw_fullscreenversion`, `draw_smallwindowversion`, `wl_display_bind('wl_ivi_shell')` |
| Input | hardware keys directly through VIM (`libjcivim_api`, `VIMC_Open` / `AddClient`) |

- No libwebsockets, no Opera or HMI dependency, no network socket (the only "connect" is
  `wl_display_connect`).
- Toggled by a hardware key through `systemHooks.sh` `SYSTEM_CPUGAUGE_TOGGLE` →
  `jci-cpugauge.sh on|off|toggle|display_off`. Flag file: `/data/integration/cpugauge_off_flag`
  (`-e` = `display_off`).

### Diagnostics menu

The key combination Mute + Music + Favourite held 3–5 s opens a number pad that takes 2-digit codes
("Invalid" on a wrong code). It is `/jci/gui/apps/diag/js/diagApp.js` ("IHU GUI Test Screen
Diagnostics", JS/HTML, `baseApp`). It looks native only because `diagApp.css` is 0 bytes
(unstyled). It renders in Opera, so it disappears with an Opera black screen.

### Factory test mode

Fully native, no Opera:

1. `autostart` reads `/data_persist/testmode.conf`, before SM starts.
2. `TDE_TRIG_TEST_MODE` → `start_test_mode.sh` kills the normal compositor and relaunches
   `wayland-viv-compositor` with `/etc/wayland/compositor_testmode.conf`.
3. It runs the native `testapp` / `testdiag` (`/jci/testapp`, `/jci/testdiag`).

Opera (`/jci/opera/opera`) never starts. No key combination writes `testmode.conf` at run time;
only `testapp` itself does.

## Verified on

Stock rootfs of firmware `cmu150_EU_74.00.324A` (ELF and script analysis).
