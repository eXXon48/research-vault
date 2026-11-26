---
tags: [cmu150, logging, syslog-ng]
---

# Syslog-ng

Stock log daemon, syslog-ng **3.3** (`@version: 3.3`), modules in `/usr/lib/syslog-ng/`
(`afsocket`, `affile`, `afprog`, `afmongodb`, `dbparser`, …). Facts below are from the stock
rootfs (fw 74.00.324A).

## Config files

- Template: `/etc/syslog-ng.conf` (rootfs, read-only).
- Live: `/data/log/syslog-ng.conf` (writable `/data`, persists across boots). It is started as
  `syslog-ng --cfgfile=/data/log/syslog-ng.conf` by `jci-log.sh start`, which runs from
  `start_normal_mode.sh` / `start_test_mode.sh`.
- `jci-log.sh` (`/jci/scripts/`, linked from `/jci/bin/`) copies the template to `/data` only
  when the live file is missing or on `jci-log.sh reset`.
- Reload: `jci-log.sh reload` = `killall -HUP syslog-ng`.

## Sources (`s_system`)

- `unix-dgram("/dev/log")`
- `file("/proc/kmsg" program-override("kernel") flags(kernel))`: the kernel log.
- `pipe("/tmp/redirlogs/stdout")` and `pipe("/tmp/redirlogs/stderr")`: stdout/stderr of the
  SM services (`program-override("Std")`, info / err).
- `internal()` (`s_syslogng`).

Every line passes `parser(p_jci)` (`db_parser`, `/etc/syslog-ng/rules/rules.xml`) and the
rewrites `r_jci_log_tstamp` and `r_nonjci`. Together they fill the `jci_*` values (`jci_zone`,
`jci_pid`, `jci_level`, `jci_file`, `jci_line`, `jci_func`, `jci_msg`, `jci_log_tstamp`).

## Destinations and paths

| destination | target | active |
|---|---|---|
| `d_ram_log` | `process-log /var/log/running_log/ 512` | yes |
| `d_error_log` | `process-log /data_persist/log/error_logs/ 2048` (errors, `f_useless` dropped) | yes |
| `d_console` | `/dev/console` (UART) | yes |
| `d_tcp` | `tcp("1.2.3.4" port(50505))`, template `t_forensic` | commented out |
| `d_buffer`, `d_message_log`, `d_dummy`, `d_usertty` | — | commented out |

- `t_default`: `${jci_log_tstamp} ${jci_zone}[${jci_pid}] ${jci_level} (${jci_file}:${jci_line} ${jci_func}) ${jci_msg}`.
- `t_forensic` adds `jci_process` and `jci_tid`. It has no `<PRI>` header, so it is not a
  syslog wire format.
- `jci-log.sh tcp set <host>` / `tcp enable` / `tcp disable` / `tcp status` switch `d_tcp`
  (the port stays 50505).
- `#flags(flow-control)` is commented out, so a stalled destination does not block the
  sources. `jci-log.sh flow-control enable` would turn it on.

## Global options (selection)

`log_msg_size(1024)`, `log_fifo_size(20000)` (in-memory output queue per destination),
`flush_timeout(500)`, `time_reopen(10)` (reconnect interval), `ts_format(iso)`, `use_dns(no)`.
The 3.3 open-source build has no disk buffer.

See also: Boot Timeline ("Boot log volume"), Partitions and Storage (`/data_persist/log/`).

## Producer side: `com_jci` log device
- Services log through the kernel module `com_jci.ko` (misc device `com.jci.cpp.common.Log`); the
  `jci` Logger process (`libjci.so`, `Logger.xml`, sink `system://`) forwards to syslog.
- Runtime attributes in `/sys/devices/platform/com.jci.cpp.common.Log/` (mode 0666): `level`
  (`Info` at boot), `enable`, `strip`, `lossless`, `pause`, `capacity` (1048576), `discard`,
  `metadata`, `statistics`, `notify`, `test`. The module also carries `<Process>`/`<Zone>`/`<Levels>`
  format strings, which suggests per-process or per-zone levels (not tested).
- The boot-time level comes from `/config/common/log/settings.txt` (`COMMON_LOG_LEVEL=Info`);
  `jci-log.sh level` rewrites it, `jci-log.sh reset` sets it back to Info.
