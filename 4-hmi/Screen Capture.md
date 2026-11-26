---
tags: [cmu150, screenshot, display, reference]
---

# Screen Capture

How to capture the composited HMI screen as a PNG, and how to record it as H.264 video.

## Overview

- **Still image:** `/usr/bin/screenshot`, a Wayland client, writes an 800×480 PNG.
- **Video:** there is no stock screen recorder. `/dev/fb0` holds the composited frame and can be
  read directly. The fastest path converts the shown `fb0` page to I420 on the IPU and encodes it on
  the VPU, at ~1 % of one core.

Related: [[Display Architecture]], [[SSH Access]], [[Custom Apps (HMI)]].

## Reference

### `/usr/bin/screenshot`

- A Wayland client: it links `libwayland-client`, `libwayland-egl` and `libwayland-cursor`, and
  uses `wl_output`.
- It connects to the `winston` compositor (wayland-viv, Vivante), grabs the composited output and
  writes `wayland-screenshot.png` into its current working directory: 800×480, 8-bit RGB PNG,
  ~330 KB.
- Exit status 0 and a non-empty PNG mean success. The grab is one-shot.
- A plain shell has no Wayland environment. The values Opera runs with (`/proc/<pid>/environ`) are
  `XDG_RUNTIME_DIR=/tmp` and `WAYLAND_DISPLAY=wayland-0` (socket `/tmp/wayland-0`).
- BusyBox `timeout` uses the old syntax, `timeout -t SECONDS CMD`. Plain `timeout 15 CMD` fails
  with "can't execute '15'".

### Framebuffers

`/sys/class/graphics`:

| Device | Name | Geometry | Format |
|---|---|---|---|
| `fb0` | `DISP3 BG` | virtual 800×960 (two 800×480 pages), stride 3200 | 32 bpp, byte order BGRX |
| `fb1` | `DISP3 FG` overlay | 800×1440 | 16 bpp |

- `/dev/fb0` holds the composited HMI frame (checked against `screenshot`).
- Content drawn on the `fb1` overlay is not in `fb0`.
- `/usr/bin/v4lcapture` and `v4lcapture_ctrl` are V4L2 camera-in (rear camera), not the HMI screen.

### Video tools in the stock rootfs

- `winston` (the compositor) carries only the `screenshooter` protocol; no `wcap` or recorder.
- GStreamer 0.10 only (`gst-launch-0.10`, `gst-inspect-0.10`). Plugins in `/usr/lib/gstreamer-0.10`:
  core elements (`filesrc`, `fdsrc`, `filesink`, `capsfilter`, `queue`, `tee`), `ffmpegcolorspace`,
  `videorate`, `videotestsrc`, `encodebin`, and the Freescale `mfw_*` set.
- `libmfw_vpu.so` (`mfw_vpu`) contains the hardware encoder `vpuenc` (i.MX6 VPU, firmware
  `/lib/firmware/vpu/vpu_fw_imx6{d,q}.bin`, `libvpu.so.4`). Caps strings: `video/x-h264`,
  `video/x-h263`, `mpeg4`, `image/jpeg`; input `video/x-raw-yuv`.
- `libmfw_gst_ipu_csc.so` (`mfw_ipucsc`): hardware colour-space conversion. It takes only YUV
  (NV12/I420) input.
- Missing: container muxers (avi, mp4, matroska), `multifilesrc`, `videoparse`, any screen source
  element (`fbdevsrc`, Wayland source), and an `ffmpeg` binary. `mfw_v4lsrc` / `v4lcapture` are
  camera-in.

### IPU task device

`/dev/mxc_ipu`, kernel 3.0.35 BSP, `include/linux/ipu.h`:

| Item | Value |
|---|---|
| `struct ipu_task` | 136 bytes |
| `IPU_CHECK_TASK` | `0xC0884901` |
| `IPU_QUEUE_TASK` | `0x40884902` |
| `IPU_ALLOC` | `0xC0044903` |
| `IPU_FREE` | `0x40044904` |

The numbers are the same as in the stock `libmfw_gst_ipu_csc.so`.

- Input format `BGR4` (`fourcc('B','G','R','4')`) gives correct colours for `fb0`; output `I420`.
- The shown page comes from `yoffset` (`FBIOGET_VSCREENINFO`). Its physical address is
  `smem_start + yoffset * line_length` (`FBIOGET_FSCREENINFO`, offsets 16 and 44 on 32-bit ARM).
- An `IPU_ALLOC` buffer can be mapped through `/dev/mxc_ipu` (offset = physical address), but the
  mapping is uncached.

### VPU library

`/usr/lib/libvpu.so.4` is imx-lib vpulib 5.4.10, firmware 2.1.8.

- The public imx-lib `vpu/vpu_lib.h` at `VPU_LIB_VERSION(5, 4, 10)` matches the stock library
  (`EncOpenParam` is 2120 bytes on 32-bit ARM).
- `libvpu` prints its errors with `printf` to stdout.
- `libvpu` is linked against the unit's glibc 2.11. A program that loads it must be a dynamic
  glibc binary; a static musl binary cannot `dlopen` it. glibc 2.11 has no `getauxval` (2.16+), and
  its plain `mmap` takes a 32-bit `off_t` (use `mmap64`).
- `vpu_Init` does not reload the firmware when the VPU already runs. `vpu_UnInit` shuts the VPU down
  only when no instance is left. Several processes can use the VPU at once.

## How to

### Take a screenshot

```sh
cd /tmp && XDG_RUNTIME_DIR=/tmp WAYLAND_DISPLAY=wayland-0 /usr/bin/screenshot
```

The result is `/tmp/wayland-screenshot.png`.

### Record H.264 with GStreamer

Writes a raw H.264 elementary stream. Colours and frame count were verified with
`ffprobe`/`ffmpeg` off the unit.

```sh
CAPS="video/x-raw-rgb,bpp=32,depth=24,endianness=4321,red_mask=65280,green_mask=16711680,blue_mask=-16777216,width=800,height=480,framerate=24/1"
(while :; do dd if=/dev/fb0 bs=1536000 count=1 2>/dev/null; done) |
  gst-launch-0.10 -q fdsrc blocksize=1536000 ! $CAPS ! ffmpegcolorspace !   video/x-raw-yuv,format=\(fourcc\)I420 ! vpuenc codec=6 bitrate=2000000 ! filesink location=/tmp/out.h264
```

- RGB to YUV runs on the CPU (`ffmpegcolorspace`), because `mfw_ipucsc` takes only YUV input. This
  is the bottleneck.
- The `dd` loop does not pace frames. `framerate=24/1` in the caps only labels them.

### Record H.264 with the IPU and VPU directly

No CPU pixel work: the IPU converts the shown `fb0` page to I420 by DMA, and the VPU encodes it
from the physical address. The CPU paces frames and copies the encoded bitstream.

VPU flow, as in imx-test `mxc_vpu_test/enc.c`:

1. `vpu_Init`.
2. `IOGetPhyMem` / `IOGetVirtMem` for the bitstream buffer.
3. `vpu_EncOpen` (`STD_AVC`, `bitRate` in kbit/s, max 32767).
4. `vpu_EncGetInitialInfo`.
5. `vpu_EncRegisterFrameBuffer`: `minFrameBufferCount` reconstruction buffers + 2 subsample
   buffers, stride a multiple of 16.
6. `ENC_PUT_AVC_HEADER` for SPS and PPS.
7. Per frame: `vpu_EncStartOneFrame` (source `FrameBuffer` = the IPU I420 buffer), then
   `vpu_IsBusy` / `vpu_WaitForInt`, then `vpu_EncGetOutputInfo` (bytes at `bitstreamBuffer`).

## Measurements

Bench unit. Rates with the HMI idle on a static screen unless noted:

| Path | Rate |
|---|---|
| `screenshot` loop | ~1.7 fps (0.6 s per PNG) |
| `dd if=/dev/fb0 bs=1536000 count=1` loop, to `/dev/null` | ~40 fps |
| raw frames over `ssh` (2.4 GHz Wi-Fi, `wlan0`) | ~1.6 MB/s = ~1 fps |
| `dd` loop piped into `lzop -1` | ~12 fps, ratio ~7× |
| `dd` loop → `fdsrc ! ffmpegcolorspace ! vpuenc codec=6` (H.264) | ~12.8 fps, ~3 KB/frame |
| same, with `queue` elements between the stages | ~14 fps |
| `ffmpegcolorspace` alone | ~38 fps |
| `vpuenc` H.264 from I420 alone | > 45 fps |
| IPU buffer read through the uncached mapping | ~44 MB/s, ~31 % of a core for 24 fps × 576000 bytes; piping into `gst-launch … vpuenc` adds ~15 %; ~18.5 fps with animations on screen |
| IPU + VPU directly, 800×480, 24 fps, 4 Mbit/s, menus and knob carousel animating | 600 of 600 frames in 25 s, none dropped, ~0.3 s CPU in total (~1 % of one core) |

Raw frames do not fit the Wi-Fi link: 24 fps × 1.5 MB = ~37 MB/s.

## Verified on

Firmware `cmu150_EU_74.00.324A`: stock rootfs for the inventory, bench unit for the measurements.
