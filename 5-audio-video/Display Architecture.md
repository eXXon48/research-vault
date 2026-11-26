---
tags: [cmu150, display, wayland, kb]
---

# KB — CMU150 Display Architecture

Stock display/compositor stack of firmware `74.00.324A`, including an ELF analysis of
`jci_cpu_gauge`. Companion to [[Audio Architecture]].

## Stack = Wayland + GENIVI IVI-shell

The display is **Wayland**, with compositor **`wayland-viv-compositor`** (Vivante/Verisilicon
GPU) and **GENIVI IVI-shell** layer management (`wl_ivi_shell`, `libwayland-ivi-*`). Clients
composite independent surfaces onto z-ordered IVI layers. There are **three** rendering
clients, and only one of them is Opera:

| Client | Kind | What it draws |
|---|---|---|
| `/jci/nativegui/svcjcinativeguictrl` (+ `svcjcinativegui.so`) | native Wayland | **static PNGs only** — boot splash, SafetyText ("don't watch while driving"), NoVideoSignal, PSMOpen (~101 images) |
| `/jci/cpugauge/jci_cpu_gauge` | native Wayland + Cairo | **live dynamic text** overlay (CPU / mem / threads / Linux version) |
| Opera / `jciMMUI` | Opera | the HMI — renders the JS/HTML apps under `/jci/gui` |

## `jci_cpu_gauge` is a standalone native Wayland client

From `DT_NEEDED` and dynsym (76 475 B, armv7 ELF):

- **NEEDED:** `libwayland-client.so.0`, `libwayland-ivi-client.so.0`,
  `libwayland-ivi-shell-client.so.1`, `libcairo.so.2`, `libjcivim_api.so` (+ libjcicommon/pc/
  pcapi, stdc++/m/pthread/c).
- **Imports:** `wl_display_connect`, `wl_proxy_marshal*`, binds `wl_compositor` / `wl_shell` /
  `wl_ivi_shell` / `wl_shm`, `wl_shm_create_pool` / `wl_shm_create_buffer` (its own
  framebuffer); draws with `cairo_*` (`cairo_image_surface_create_for_data`, `cairo_show_text`,
  move/rect/stroke).
- **Strings:** `ws_tm_draw`, `draw_fullscreenversion`, `draw_smallwindowversion`,
  `wl_display_bind('wl_ivi_shell')`.
- **No** libwebsockets, **no** Opera/HMI dependency, **no** network socket (the only "connect" is
  `wl_display_connect`). Reads hard-keys directly via VIM (`libjcivim_api`, `VIMC_Open` /
  `AddClient`).
- Toggled by hard-key via `systemHooks.sh` `SYSTEM_CPUGAUGE_TOGGLE` → `jci-cpugauge.sh
  on|off|toggle|display_off` (flag `/data/integration/cpugauge_off_flag`; `-e` = display_off).

## The diag "dev menu" is Opera, not native

The community combo (Mute + Music + Fav/Star 3–5 s → number-pad, 2-digit codes, "Invalid") is
`/jci/gui/apps/diag/js/diagApp.js` ("IHU GUI Test Screen Diagnostics", JS/HTML, baseApp). It
looks native only because `diagApp.css` is **0 bytes** (unstyled). It renders in Opera, so
it disappears with an Opera black screen.

## Factory test mode is fully native (no Opera)

`autostart` reads `/data_persist/testmode.conf`; `TDE_TRIG_TEST_MODE` → `start_test_mode.sh`
kills the normal compositor, relaunches `wayland-viv-compositor` with
`/etc/wayland/compositor_testmode.conf`, and runs native `testapp` / `testdiag`
(`/jci/testapp`, `/jci/testdiag`). Opera (`/jci/opera/opera`) never starts. Pre-SM dispatch.
Nothing writes `testmode.conf` via a key-combo at runtime (only `testapp` itself).

## Surfaces independent of Opera

A native Wayland IVI-shell client has its own IVI layer above the HMI layer, its own `wl_shm`
buffer and Cairo text, so it keeps drawing when Opera shows a black screen. `jci_cpu_gauge`
is the on-device example: lightweight, hard-key toggled, VIM-input capable,
Opera-independent. It needs the compositor and VIM, so it exists only in the SM era (after the
service manager has started them), not earlier in boot.

See also [[Audio Architecture]].
