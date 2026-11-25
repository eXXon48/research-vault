# research-vault

Reference documentation of the Mazda Connect CMU150 head unit (MX-5 ND Gen1, firmware
cmu150_EU_74.00.324A): hardware, partitions, boot chain, services, HMI, display and audio.
The notes are an [Obsidian](https://obsidian.md) vault, but every file is plain Markdown and
reads fine on GitHub.

Part of [cmu-REvised](https://github.com/eXXon48/cmu-REvised).

## Where to start

| Path | Contents |
|---|---|
| [`CMU150.md`](CMU150.md) | Index of the platform notes |
| `cmu150-re/` | Hardware, partitions, boot, users, SSH, clock, HMI, display, audio |
| `jci-ui/` | HMI framework reference: architecture, messaging bus, app anatomy, how to extend it |
| [`BusyBox.md`](BusyBox.md), `commands/` | Reference of the BusyBox applets on the unit |

## Scope

Observations and descriptions produced by reverse engineering, for interoperability and repair.
No firmware, binaries, keys or map data are included; see [`NOTICE.md`](NOTICE.md).
