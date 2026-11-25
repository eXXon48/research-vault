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

Framebuffers (`/sys/class/graphics`): `fb0` = `DISP3 BG`, virtual 800x960 (two 800x480 pages),
32 bpp, stride 3200, byte order BGRX; `fb1` = `DISP3 FG` overlay, 800x1440, 16 bpp.
`/dev/fb0` holds the composited HMI frame (checked against `screenshot`). Content drawn on the
`fb1` overlay is not in `fb0`.

## Video: measured (bench, HMI idle on a static screen)
| Path | Rate |
|---|---|
| `screenshot` loop | ~1.7 fps (0.6 s per PNG) |
| `dd if=/dev/fb0 bs=1536000 count=1` loop, to `/dev/null` | ~40 fps |
| raw frames over `ssh` (2.4 GHz Wi-Fi, `wlan0`) | ~1.6 MB/s = ~1 fps |
| `dd` loop piped into `lzop -1` | ~12 fps, ratio ~7x on a static screen |
| `dd` loop, `fdsrc ! ffmpegcolorspace ! vpuenc codec=6` (H.264) | ~12.8 fps, ~3 KB/frame on a static screen |
| same with `queue` elements between the stages | ~14 fps |

Working pipeline (writes a raw H.264 elementary stream; colours and frame count verified with
`ffprobe`/`ffmpeg` off-unit):

```sh
CAPS="video/x-raw-rgb,bpp=32,depth=24,endianness=4321,red_mask=65280,green_mask=16711680,blue_mask=-16777216,width=800,height=480,framerate=24/1"
(while :; do dd if=/dev/fb0 bs=1536000 count=1 2>/dev/null; done) |
  gst-launch-0.10 -q fdsrc blocksize=1536000 ! $CAPS ! ffmpegcolorspace !   video/x-raw-yuv,format=\(fourcc\)I420 ! vpuenc codec=6 bitrate=2000000 ! filesink location=/tmp/out.h264
```

Notes:
- `mfw_ipucsc` takes only YUV (NV12/I420) input, so RGB to YUV runs on the CPU (`ffmpegcolorspace`).
- Alone, each stage is faster than the whole pipeline: `ffmpegcolorspace` ~38 fps, `vpuenc` H.264
  from I420 > 45 fps. The bottleneck is the CPU colour conversion.
- The `dd` loop does not pace frames: `framerate=24/1` in the caps only labels them.
- Raw frames do not fit the Wi-Fi link: 24 fps x 1.5 MB = ~37 MB/s.
