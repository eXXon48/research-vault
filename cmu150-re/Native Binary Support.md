---
tags: [cmu150, native, arm, kernel]
---

# Native Binary Support

What kind of native executable the CMU150 runs, and which input-injection tools the stock
userland lacks. Firmware `74.00.324A`. Related: [[Hardware and Firmware]].

- Kernel: Linux 3.0.35 (ARM, i.MX6-class). The userland is glibc-based with BusyBox 1.19.2.
- Statically linked ARM executables (`arm-unknown-linux-musleabi`, soft-float, fixed-address
  `ET_EXEC`, no `.note.ABI-tag`) run on the unit; musl binaries have no glibc minimum-kernel
  gate on 3.0.35.
- The stock image has no `evemu`, `uinput`, `evtest` or `getevent` tools. BusyBox provides
  only `devmem` ([[devmem]]).
- `/dev/uinput` exists (root only, see [[CMU150 Users and Privileges]]); creating an input
  device through it requires a native binary.
