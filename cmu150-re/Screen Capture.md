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

## Video (stock rootfs inventory)
No stock screen recorder exists. Found in the stock rootfs (`74.00.324A`):
- `winston` (compositor) carries only the `screenshooter` protocol; no `wcap` / recorder.
- GStreamer **0.10** only (`gst-launch-0.10`, `gst-inspect-0.10`). Plugins in `/usr/lib/gstreamer-0.10`:
  core elements (`filesrc`, `fdsrc`, `filesink`, `capsfilter`, `queue`, `tee`), `ffmpegcolorspace`,
  `videorate`, `videotestsrc`, `encodebin`, and the Freescale `mfw_*` set.
- `libmfw_vpu.so` (`mfw_vpu`) contains a **hardware encoder `vpuenc`** (i.MX6 VPU, firmware
  `/lib/firmware/vpu/vpu_fw_imx6{d,q}.bin`, `libvpu.so.4`). Caps strings in the plugin:
  `video/x-h264`, `video/x-h263`, `mpeg4`, `image/jpeg`; input `video/x-raw-yuv`.
- `libmfw_gst_ipu_csc.so` (`mfw_ipucsc`): hardware colour-space conversion.
- No container muxer (no avi/mp4/matroska mux), no `multifilesrc`, no `videoparse`, no screen
  source element (no `fbdevsrc`, no Wayland source). `mfw_v4lsrc` / `v4lcapture` are camera-in.
- No `ffmpeg` binary.

Consequences:
- The screen reaches a pipeline only through `screenshot` (one PNG per call) or raw `/dev/fb0`
  reads; whether `/dev/fb0` holds the composited HMI frame is not verified.
- A `vpuenc` pipeline would write a raw elementary stream (e.g. `.h264`) via `filesink`; muxing
  happens off-unit. Untested on the unit.
- Frame rate of a `screenshot` loop: not measured.
