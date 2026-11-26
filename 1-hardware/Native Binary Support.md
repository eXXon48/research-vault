---
tags: [cmu150, native, arm, kernel, toolchain, uinput]
---

# Native Binary Support

What kind of native executable the CMU150 runs, how to build one, and how `uinput` behaves
on the unit.

## Overview

The unit runs soft-float ARM EABI binaries on Linux 3.0.35. The rootfs has runtime libraries
only, with no headers or static libraries, so nothing can be built or linked on the unit.
Statically linked musl executables built on a PC run as they are. Current static glibc
binaries do not: they require a newer kernel. The stock image has no input-injection tools,
so a virtual input device through `/dev/uinput` needs your own native binary.

Related: [[Hardware and Firmware]].

## Target ABI
- Kernel: Linux 3.0.35 (`/lib/modules/3.0.35/`). SoC i.MX6, Cortex-A9 (armv7, VFPv3, NEON). `/proc/cpuinfo`: `ARMv7 Processor rev 10 (v7l)`, `Features: swp half thumb fastmult vfp edsp neon vfpv3`.
- Userland: glibc 2.11.1 with BusyBox 1.19.2.
- Dynamic linker: `/lib/ld-linux.so.3` (not the `-armhf` variant). There is no
  `arm-linux-gnueabihf` directory, so the userland is soft-float EABI (`gnueabi`).
- Stock binaries carry ELF `e_flags` = `0x05000002` (EABI v5, float-ABI bit unset).
- The rootfs ships only runtime libraries (`libc.so.6` → `libc-2.11.1.so`, libm, libpthread,
  libdl, librt, `libgcc_s.so.1`). It has no crt objects, no `libc.a` and no headers, so
  nothing on the unit can be linked against.

## What runs
- Statically linked musl executables (`arm-linux-musleabi`, soft-float, `-static -no-pie`,
  fixed-address `ET_EXEC`, no `.note.ABI-tag`) run on the unit. musl has no minimum-kernel
  gate. A static binary that uses no floating point is independent of the float ABI.
- Static glibc binaries from current toolchains are stamped with a minimum kernel (for
  example `for GNU/Linux 5.4.0`). On 3.0.35 they abort with `FATAL: kernel too old`.
- Static-PIE (`ET_DYN` without an interpreter) is unreliable on this kernel generation. Use
  `ET_EXEC`.

## Building
Toolchain: `arm-linux-musleabi-cross` from <https://musl.cc/> (gcc 11.2.1). Example:

```sh
arm-linux-musleabi-gcc -static -no-pie -fno-pie -O2 -Wall -o probe probe.c
```

A small C probe builds to about 86 KB.

## uinput
- The stock image has no `evemu`, `uinput`, `evtest` or `getevent` tools. BusyBox provides
  only `devmem` ([[devmem]]). Creating an input device through `/dev/uinput` requires a
  native binary.
- `uinput.ko` is a loadable module: `/lib/modules/3.0.35/kernel/drivers/input/misc/uinput.ko`.
  On a running unit it is loaded at boot and `/dev/uinput` exists (`c 10,223`, root only, see
  [[Users and Privileges]]). The node is not present in the rootfs image because
  devtmpfs populates `/dev` at runtime.
- Kernel 3.0.35 supports only the legacy uinput API. `UI_DEV_SETUP` and `UI_GET_SYSNAME`
  (kernel 4.5+) do not exist. Sequence: `open("/dev/uinput", O_WRONLY|O_NONBLOCK)` →
  `UI_SET_EVBIT` / `UI_SET_KEYBIT` / `UI_SET_ABSBIT` → `write(struct uinput_user_dev)` →
  `UI_DEV_CREATE` → … → `UI_DEV_DESTROY`.
- No ioctl returns the new device's node name. Find the node by its name in
  `/proc/bus/input/devices`.
- With stock nodes `event0`–`event5` present, a created device appears as `event6` and is
  removed again on `UI_DEV_DESTROY`.

## Verified on

Firmware `cmu150_EU_74.00.324A`.
