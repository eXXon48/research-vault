---
tags: [cmu150, storage, flashfx, nand, kernel, boot]
---

# FlashFX Read Path

How the root filesystem is read from NAND through the Datalight FlashFX driver, what each read
costs, and why the read thread `ffxread01` is the largest CPU user during boot.

## Overview

The root filesystem (`ffx01`) sits on a Micron SLC NAND chip behind Datalight FlashFX Tera. One
kernel thread per disk, `ffxread01` for the root, serves every read request synchronously, one
4 KiB page at a time. Each page takes three DMA chains and four interrupts. The flash translation
layer also loads allocation maps for each region it touches; during boot these map loads take
about half of the read thread's time. The net cost is about 74 ms of kernel CPU per MB read. The
driver is closed, so the only lever is to read fewer bytes.

Related: [[Partitions and Storage]], [[Boot Chain and Fail-Safe]] (module ABI),
[[Boot Timeline]], [[Boot Speed Levers]].

## Reference

### Driver

Datalight FlashFX Tera v2.1.1 (build 2128DF):

| Module | Role |
|---|---|
| `ffxblk.ko` | block driver |
| `flashfx.ko` | VBF flash translation layer + NAND control |
| `ffxos.ko` | OS services + i.MX6 GPMI/BCH hook |

- Copies in the rootfs: `/lib/modules/3.0.35/kernel/drivers/datalight/{flashfx,ffxos,ffxblk}/`.
  The normal boot loads byte-identical copies from the initramfs; the fail-safe initramfs
  (`mtd7`) has its own ([[Boot Chain and Fail-Safe]]).
- All modules are built with `-pg` (`__gnu_mcount_nc` at every function entry). The kernel has
  dynamic ftrace (`ftrace_make_nop`, `ftrace_module_notify` in `/proc/kallsyms`), and each module
  has `__mcount_loc`, so these calls are patched to NOPs at load. `-pg` costs about nothing at
  runtime.

### NAND chip

From the ONFI parameter data as decoded by `flashfx.ko`, read from kernel memory:

| | |
|---|---|
| Part | Micron `MT29F8G08ABABAWP` (JEDEC ID `0x2C`) |
| Type | 8 Gbit SLC, x8, 1 LUN |
| Geometry | 4096-byte page + 224-byte spare, 128 pages per block, 2048 blocks |
| Timing | tR max 25 µs, tPROG max 500 µs, tBERS max 3 ms |

### GPMI timing

- The timing is fixed in `flashfx.ko`, not read from the chip. `FfxNandIfcInit` calls
  `FfxHookNTMX6SetTiming` with constants: data setup 3, data hold 2, address setup 1 (GPMI
  cycles). On a running unit `TIMING0 = 0x00010203`. The exported `FfxHookNTMX6CalculateTiming`
  (computed timing) is not used.
- FlashFX does not set the GPMI clock (`clk_get_rate` only). `CCM_CS2CDR = 0x000A0148` selects
  PLL3 (480 MHz, `ANALOG_PLL_USB1 = 0x80003040`) / 3 = 160 MHz.
- One data cycle is 5 × 6.25 ns = 31.25 ns: ~32 MB/s on the bus, about ONFI timing mode 4.
  `CTRL1 = 0x00C6C00C` (DLL enabled).
- A 4 KiB page + spare transfers in ~135 µs. With tR ≤ 25 µs, the NAND part of a ~410 µs request
  is ~160 µs; the rest is driver, DMA-chain and interrupt overhead.

### The read thread

`ffxblk` creates one `ffxreadNN` and one `ffxwriteNN` kthread per disk. `ffxread`:

1. loops on `blk_fetch_request`, and sleeps on a semaphore when the queue is empty;
2. serves each request synchronously in its own context: one request, one page at a time, no
   overlap.

Per NAND page (`PageRead`, `flashfx.ko` 0x24b4):

1. `FfxNandIfcSendCommand`: READ command `00h` + address. One GPMI DMA chain.
2. `FfxNandIfcSendCommand`: `30h`. A second chain.
3. `FfxNandIfcReadyWait` → `FfxHookNTMX6ReadyWait`: **busy-polls** the GPMI status register
   (offset `0xb0`, bit 24) with no sleep until the chip is ready.
4. `FfxNandIfcReadPage`: a third chain, DMA of the page with BCH hardware ECC
   (`FfxHookNTMX6DMAStart`/`DMAFinish`). It waits on both the DMA and the `BCHIrqHandler`
   completion, then `dma_unmap_sg`.

Each chain is `AddPIOBlock` → `DMAStart` → `DMAFinish`, which sleeps in
`wait_for_completion_timeout`. Result per page: 3 `mxs-dma` interrupts + 1 `GPMI NAND BCH`
interrupt, each with a tasklet and a sleep/wake of `ffxread`. `/proc/interrupts` agrees:
`mxs-dma` 375 549, `GPMI NAND BCH` 124 152 (ratio 3.0), `TASKLET` 938 918 on cpu0. For
comparison, mainline `gpmi-nand` puts command, address, wait-for-ready and the data read in one
chain: one interrupt per page.

A plain 4 KiB read is one NAND page read. Separate spare-area reads (`FfxNandIfcReadAux`) happen
only inside allocation-map loads.

### Locking

- Every hardware access of the NAND device goes through `FfxDevIOProlog`/`FfxDevIOEpilog`
  (`FfxDevIORequest`, `GenericRead`, `GenericWrite`, the `FfxDevNand*Block/PageStatus` calls).
- These take the per-device DCL mutex `pDev[6]` (`pDev + 0x18`). The DCL mutex object wraps a
  kernel `struct mutex` at its offset `0x14`, taken by `FfxDclOsMutexAcquire` → `mutex_lock`.
- There is one FIM device. `aFIMHandles` (`flashfx.ko` `.bss + 0x1c8`, read by `FfxDevHandle`)
  holds `pDev`.
- Holding `pDev[6]` gives exclusive use of the GPMI controller against `ffxread`/`ffxwrite` and
  the compaction thread.
- Section bases: `/sys/module/flashfx/sections/`. Function symbols, local ones included, are in
  `/proc/kallsyms`. Data symbols are not (`CONFIG_KALLSYMS_ALL` off).

### Region cache (allocation maps)

From `flashfx.ko` disassembly:

- `FfxVbfCreate` creates the cache with `N = clamp(R * 50 / 100, min(R, 2), 255)` slots of
  5 588 bytes, where `R` is the region count (`vbf + 0x28`). No option controls it.
- The cache object is at `vbf + 0x2038`: slot count at `+8`, slot pointers from `+0x10c`. A slot
  holds the region number at `+8`, a refcount at `+4` and a last-use time stamp at `+0x14`. `N` is
  also stored at `vbf + 0x2034`.
- Lookup and least-recently-used replacement are in the function at `flashfx.ko` 0x81c4. A miss
  loads the region (`FfxMediaMgrReadLnuAllocations`).
- `FfxVbfHandle(disk)` (exported) returns a pointer to the `vbf` pointer.

Values on the unit (read-only, ~510 s uptime):

| Disk | `R` | `N` |
|---|---|---|
| `ffx00` | 24 | 12 |
| `ffx01` | **978** | **255** (the cap; 50 % would be 489), all slots in use, regions 0–977 present |

A region of `ffx01` is ~0.93 MB, so the cache covers ~26 % of the disk (~237 MB). A boot reads from
a ~448 MB used root filesystem.

### Queue and kernel

- `ffx01` queue: `noop` scheduler, `read_ahead_kb` 128, `rotational` 1, 4096-byte logical blocks.
- Kernel `.config` is embedded in `/lib/modules/3.0.35/kernel/kernel/configs.ko` (gzip after the
  `IKCFG_ST` marker; extractable offline): `SMP`, `PREEMPT`, `AEABI`, `MXS_DMA`,
  `FUNCTION_TRACER`, `DYNAMIC_FTRACE`; no `DEBUG_MUTEXES`/lockdep, no `MTD_NAND_GPMI_NFC`.
- The kernel has `squashfs` built in (`/proc/filesystems`).

## How to: trace the read path with ftrace

1. `mount -t debugfs none /sys/kernel/debug`.
2. Filter `Ffx*` + `mxs_dma*` (440 functions). `available_filter_functions` lists module functions
   without the `[module]` suffix.
3. Set the tracer to `function_graph`.

## Measurements

### Cost per MB

Warm boot, uptime ~1100 s: `ffxread01` used 11.18 s of system time for 150.8 MB read from `ffx01`
(`/proc/diskstats`). That is **~74 ms CPU per MB**, or ~13.5 MB/s if CPU-bound. 123 k
`GPMI NAND BCH` interrupts.

### Traced 1 MiB cold read

`dd bs=64k` from `/dev/ffx00`, 0.30 s with tracing on. `/proc/interrupts` delta: `mxs-dma` +1647,
`GPMI NAND BCH` +547 for 549 NAND page reads (256 data pages of 4 KiB + 291 reads of the
allocation-map load). That is exactly 3 DMA + 1 BCH interrupts per `FfxNandChipPageRead`.
Inclusive times, tracing overhead included (~2 µs per traced call):

| Function | Calls | Total | Per call |
|---|---|---|---|
| `FfxDriverReadData` (whole request) | 288 | 278 ms | 966 µs |
| `FfxNandChipPageRead` (one NAND page) | 549 | 230 ms | 420 µs |
| `FfxMediaMgrReadLnuAllocations` (FTL map load) | 4 | 149 ms | 37 ms |
| `FfxNandIfcReadPage` (data DMA + BCH) | 291 | 90 ms | 309 µs |
| `FfxFmlNandReadTags` (inside the map load) | 6 | 81 ms | 13.5 ms |
| `FfxNandIfcSendCommand` (`00h`, `30h`) | 1098 | 68 ms | 62 µs |
| `FfxNandIfcReadAux` (spare-area reads, map load) | 256 | 43 ms | 167 µs |
| `FfxHookNTMX6DMAFinish` | 3294 | 113 ms | 34 µs |
| `FfxNandIfcReadyWait` | 551 | 3.3 ms | 6 µs |

- Half of the cold read is the allocation-map load (4 calls, 149 ms), not data. The 4 map loads
  issued 256 spare reads.
- Per data page, the two command chains cost ~124 µs and the data chain ~309 µs.
- `ReadyWait` is short (6 µs): the chip is ready by the time the `30h` chain completes.

### Warm and cold regions

Raw reads of `/dev/ffx01p1`, 2 MiB each, same tracing, ~15 min after boot:

| Region | Time | Page reads | Map loads | Per 4 KiB request |
|---|---|---|---|---|
| offset 0 (read during boot) | 0.38 s | 511 | 0 | 674 µs |
| offset 300 MiB | 0.48 s | 774 | 4 (150 ms) | 896 µs |

- The allocation map of the region read at boot stays cached. A region not read since boot pays
  4 map loads (37 ms each, mostly 256 spare-area reads) per 2 MiB, a third of its read time.
- With a warm map a 4 KiB request costs ~674 µs traced: `FfxNandChipPageRead` 443 µs (two command
  chains ~133 µs, data chain ~291 µs), plus ~230 µs of FTL/driver code above it. Tracing inflates
  that last part most, because it is many short calls.

### During boot

Trace of `FfxDriverReadData` + `FfxMediaMgrReadLnuAllocations` only, from 7.9 s (a
`start_normal_mode.sh` hook) to 45 s. Boot metrics were unchanged by it.

| | |
|---|---|
| Requests served | 24 747 of 4 KiB (~97 MB) |
| `ffxread01` own time | **18.8 s** |
| Allocation-map loads | **744, 8.9 s** (median 9.7 ms, p90 16.6 ms, max 101 ms): one per ~130 KB read, 47 % of the thread's time |
| Data alone | ~410 µs per 4 KiB |
| Busy time 8–16 s | 1.3–1.6 s of every 2 s |

Before 7.9 s, `ffx01` had served 564 requests (15.9 MB), but the BCH interrupt count was already
32 381 (≈8× the data pages). Mount and map reads dominate that window too.

Region-cache churn: a read-only sampler read the `ffx01` cache slots every 20 ms from 7.9 s to
~53 s (one boot).

- 106 regions were already cached at 7.9 s.
- 201 region loads followed: 188 first loads and 13 reloads of an evicted region (first reload
  at +25 s). The 255-slot cap causes little re-reading during boot.
- The same boot shows 771 `FfxMediaMgrReadLnuAllocations` calls (7–10 ms each), about 3.8 per
  region load: one region load reads several map pieces in a row.

The map cost at boot is the first load of each region touched, not cache thrashing.

### Bytes read per process

`/proc/<pid>/io` `read_bytes` at ~1044 s (includes `mmcblk0`):

| Process | MB |
|---|---|
| `natp` (starts ~46 s) | 40.7 |
| `opera` | 24.9 |
| `jciMMUI` | 11.0 |
| `init` | 7.8 |
| SM | 5.1 |
| `jciVBS` | 4.9 |
| `vim_app` | 4.5 |
| `jciUSBMGR` | 4.2 |
| `usbms` | 4.0 |
| `carplayd` | 3.9 |
| `bds` | 3.4 |
| `gracenote_server` | 2.9 |

### Raw sequential read

64 KiB blocks, 32 MiB, cold: eMMC `mmcblk0p1` 16.2 MB/s, FlashFX `ffx01p1` 8.8 MB/s.

## Verified on

Firmware `cmu150_EU_74.00.324A`, legacy board.
