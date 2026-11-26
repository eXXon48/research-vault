---
tags: [cmu150, media, usb, bluetooth, gracenote]
---

# Media Album Art

Where the CMU150 gets album art for USB and Bluetooth media, which image formats work, and how
the Gracenote service affects it.

## Overview

For USB media, the RACE media engine reads the picture embedded in the audio file. Only when that
fails does it fall back to Gracenote. Bluetooth media gets art only from Gracenote. Gracenote has
no reconnect logic: if `gracenote_server` is not up when a media process starts, that process
gets no Gracenote art for the rest of the boot.

Related: [[Audio Architecture]], [[Boot Timeline]].

## Reference

### USB media (RACE)

- Process `/jci/usbm/usbms` with `/usr/lib/librace_interface.so` (RACE).
- First choice: the picture embedded in the file (`race_get_albumart_info` /
  `race_read_albumart`: offset, size and MIME type of the embedded picture).
- Fallback: Gracenote (`Failed to retrieve Albumart from device, trying to get Gracenote Albumart`
  → `GRACENOTE_CoverArtLookup`).
- The art is saved as a BMP under `/tmp/albumart` and announced with the `com.jci.usbms`
  `AlbumArtReady` signal.
- RACE also uses Gracenote for metadata lookup (`GRACENOTE_DBLookup`), "More Like This" playlists
  (`race_gnw_get_morelikethis`) and a collection DB (`/tmp/gracenotedb/`, saved to
  `/data/gracenotedb_save/`).
- Failure strings: `Gracenote support is disabled`, `Can not connect to gracenote server.`,
  `Gracenote wrapper init failed`. No reconnect logic is visible in the strings.
- In `sm.conf`, `jciUSBMS` and `jciUSBM` do not depend on `gracenote_server`. They depend only on
  `devicemanager`, and on `jciUSBMS`/`jciBLMSettings`/`audio_manager`/`jciRM`. `jciBTMUSIC` does
  depend on `gracenote_server`.

### Bluetooth media

`bds`, `svcjcibtmusic.so`: no AVRCP 1.6 / BIP cover-art strings. Bluetooth art comes only from
Gracenote; otherwise the log shows `No gracenote update. No album art.`

### What shows on a USB stick (FAT32)

| Track | Shown |
|---|---|
| embedded ID3v2 `APIC`, baseline JPEG 160×160 | yes |
| embedded `APIC`, progressive JPEG 457×639 | no: `race_dbm_get_albumart_info ... Marker FFC2 found`, then `error=0x313` |
| `Folder.jpg` / `AlbumArt*.jpg` next to the tracks, nothing embedded | no: RACE reads only the embedded picture |
| no picture | no |

Each failure then tries the Gracenote fallback (`race_gnw_get_albumart`). Progressive JPEG is not
supported.

### When Gracenote starts late

- If `gracenote_server` starts after the boot-time `usbms` (~21 s), RACE in that process logs
  `Gracenote wrapper init failed. Error=0x800`. Every later fallback fails with
  `race_gnw_get_albumart ... Recieved NULL Pointers`. Playback uses this process.
- A stick inserted later starts a separate indexing process, which connects to Gracenote normally.
- BTMUSIC that starts without Gracenote logs `ServiceInit) Gracenote Connect failed` and does not
  retry. Bluetooth tracks get title and artist but no art until the next boot.

## Verified on

Firmware `cmu150_EU_74.00.324A`; strings from the stock image, behaviour on a bench unit.
