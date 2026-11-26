---
tags: [cmu150, logging, syslog-ng]
---

# Syslog-ng

How log lines get from services and the kernel to the log files and the UART, and where the
configuration lives.

## Overview

The stock log daemon is syslog-ng **3.3** (`@version: 3.3`), with modules in
`/usr/lib/syslog-ng/` (`afsocket`, `affile`, `afprog`, `afmongodb`, `dbparser`, …).

```
services ──com_jci.ko──> jci Logger ──┐
SM service stdout/stderr (FIFOs) ─────┼──> syslog-ng ──> RAM log  /var/log/running_log/
kernel (/proc/kmsg), /dev/log ────────┘               ├─> error log /data_persist/log/error_logs/
                                                      └─> UART /dev/console
```

The live configuration is on writable `/data` and survives reboots. The template on the read-only
rootfs is copied only when the live file is missing or on reset.

Related: [[Boot Timeline]] ("Boot log"), [[Partitions and Storage]] (`/data_persist/log/`),
[[SM Service Control (smctl)]].

## Reference

### Configuration files

| File | Role |
|---|---|
| `/etc/syslog-ng.conf` | template, rootfs, read-only |
| `/data/log/syslog-ng.conf` | live configuration, writable `/data`, persists across boots |

- `jci-log.sh start` starts the daemon as `syslog-ng --cfgfile=/data/log/syslog-ng.conf`. It runs
  from `start_normal_mode.sh` and `start_test_mode.sh`.
- `jci-log.sh` lives in `/jci/scripts/` and is linked from `/jci/bin/`. It copies the template to
  `/data` only when the live file is missing, or on `jci-log.sh reset`.

### Sources (`s_system`)

| Source | Content |
|---|---|
| `unix-dgram("/dev/log")` | syslog clients |
| `file("/proc/kmsg" program-override("kernel") flags(kernel))` | kernel log |
| `pipe("/tmp/redirlogs/stdout")`, `pipe("/tmp/redirlogs/stderr")` | stdout/stderr of SM services (`program-override("Std")`, levels info / err) |
| `internal()` (`s_syslogng`) | syslog-ng's own messages |

Every line passes `parser(p_jci)` (`db_parser`, `/etc/syslog-ng/rules/rules.xml`) and the rewrites
`r_jci_log_tstamp` and `r_nonjci`. Together they fill the `jci_*` values: `jci_zone`, `jci_pid`,
`jci_level`, `jci_file`, `jci_line`, `jci_func`, `jci_msg`, `jci_log_tstamp`.

### Destinations

| Destination | Target | Active |
|---|---|---|
| `d_ram_log` | `process-log /var/log/running_log/ 512` | yes |
| `d_error_log` | `process-log /data_persist/log/error_logs/ 2048` (errors; `f_useless` dropped) | yes |
| `d_console` | `/dev/console` (UART) | yes |
| `d_tcp` | `tcp("1.2.3.4" port(50505))`, template `t_forensic` | commented out |
| `d_buffer`, `d_message_log`, `d_dummy`, `d_usertty` | — | commented out |

### Templates

- `t_default`:
  `${jci_log_tstamp} ${jci_zone}[${jci_pid}] ${jci_level} (${jci_file}:${jci_line} ${jci_func}) ${jci_msg}`.
- `t_forensic` adds `jci_process` and `jci_tid`. It has no `<PRI>` header, so it is not a syslog
  wire format.

### Global options (selection)

| Option | Value |
|---|---|
| `log_msg_size` | 1024 |
| `log_fifo_size` | 20000 (in-memory output queue per destination) |
| `flush_timeout` | 500 |
| `time_reopen` | 10 (reconnect interval) |
| `ts_format` | iso |
| `use_dns` | no |

- `#flags(flow-control)` is commented out, so a stalled destination does not block the sources.
- The 3.3 open-source build has no disk buffer.

### Producer side: the `com_jci` log device

- Services log through the kernel module `com_jci.ko` (misc device `com.jci.cpp.common.Log`). The
  `jci` Logger process (`libjci.so`, `Logger.xml`, sink `system://`) forwards the lines to syslog.
- Runtime attributes are in `/sys/devices/platform/com.jci.cpp.common.Log/` (mode 0666):

  | Attribute | Note |
  |---|---|
  | `level` | `Info` at boot |
  | `capacity` | 1048576 |
  | `enable`, `strip`, `lossless`, `pause`, `discard`, `metadata`, `statistics`, `notify`, `test` | — |

- The boot-time level comes from `/config/common/log/settings.txt` (`COMMON_LOG_LEVEL=Info`).

## How to

| Task | Command |
|---|---|
| Reload the configuration | `jci-log.sh reload` (= `killall -HUP syslog-ng`) |
| Restore the template | `jci-log.sh reset` (also sets the log level back to Info) |
| Change the boot-time log level | `jci-log.sh level` (rewrites `settings.txt`) |
| Forward over TCP | `jci-log.sh tcp set <host>`, `tcp enable`, `tcp disable`, `tcp status` (port stays 50505) |
| Turn on flow control | `jci-log.sh flow-control enable` |

## Verified on

Stock rootfs of firmware `cmu150_EU_74.00.324A`.

## Open questions

- The module carries `<Process>`/`<Zone>`/`<Levels>` format strings. Whether it supports
  per-process or per-zone levels is not tested.
