---
tags: [cmu150, screenshot, display, reference]
---

# Screen Capture

How the composited HMI screen can be captured as a PNG on the CMU150. Firmware `74.00.324A`.
Related: [[KB - CMU150 Display Architecture]], [[SSH Access]].

## The tool
`/usr/bin/screenshot` is a **Wayland client** (links `libwayland-client/egl/cursor`, uses
`wl_output`). It connects to the `winston` (wayland-viv / Vivante) compositor, grabs the
composited output and writes **`wayland-screenshot.png`** into its current working directory
(800×480, 8-bit RGB PNG, ~330 KB).

## Environment it needs
A plain shell has no Wayland environment. The values Opera runs with
(`/proc/<pid>/environ`) are:
- `XDG_RUNTIME_DIR=/tmp`
- `WAYLAND_DISPLAY=wayland-0` (socket `/tmp/wayland-0`)

Example invocation:

```sh
cd /tmp && XDG_RUNTIME_DIR=/tmp WAYLAND_DISPLAY=wayland-0 /usr/bin/screenshot
```

## Facts
- Output is `wayland-screenshot.png` in the process CWD; exit status 0 and a non-empty PNG
  mean success.
- BusyBox `timeout` uses the **old** syntax (`timeout -t SECONDS CMD`); plain
  `timeout 15 CMD` fails with "can't execute '15'". The grab is one-shot.
- `/dev/fb0` (800×480×32bpp) is the raw framebuffer, but the Vivante GPU composites via the
  compositor, so the `screenshot` client is the reliable capture path.
- `/usr/bin/v4lcapture` / `v4lcapture_ctrl` are **V4L2 (camera-in / rear cam)**, not the HMI
  screen.
- Other observability surfaces: logs at `/var/log/running_log/log`, and a live JS console
  through the HMI injection hook ([[Custom Apps (HMI)]]).
- `/var/log/running_log/log` rotates about once a minute after boot into
  `log_0000.tar.gz` … `log_0004.tar.gz` in the same directory (gzip tar, oldest first). Lines from
  HMI start are in the archives a few minutes later:
  `tar -xzOf log_0000.tar.gz | grep -a <text>`. GUI log lines carry the tag
  `GUI_<APP>[pid] <Level> (<file>:<line> )`.
