---
tags: [cmu150, sm, services, reference]
---

# SM Service Control (smctl)

How to start, stop and query System Manager (SM) services from a shell, and how SM reacts to
stops, crashes and exits.

## Overview

The System Manager `/jci/sm/sm` reads `/jci/sm/sm.conf` (or `sm_WCP.conf`) at boot and
supervises the `<service>` elements listed there. `/jci/sm/smctl` is its command-line client.
SM reads its configuration only once, at boot. smctl always exits 0, so the result has to be
read from its output.

Related: [[SM Service Graph]], [[Boot Timeline]], [[Syslog-ng]].

## Reference

### Options

| Option | Meaning |
|---|---|
| `-l -w -n <svc>` | launch and wait |
| `-s -w -n <svc>` | stop and wait |
| `-r -n <svc>` | restart |
| `-g -t info -n <svc>` | print service info |

### Output

smctl exits 0 even when the command failed. The result is only in the text output:

| Case | Output |
|---|---|
| unknown service, info | `ERROR:smctl_get_service_info_command_cb [STATUS_NOT_FOUND] <svc>` |
| unknown service, launch | `smctl_start_service_command_cb <svc> {107:Not Found}` and `WARNING:Starting service <svc> failed. Check if service with this name exists` |
| known service, info | `[STATUS_SUCCESS] <svc> [6:RUNNING]` or `[9:STOPPED]`, then a `<service … state="RUNNING\|STOPPED" … retry_count="<used>/<max>" pid="…"/>` element |

### SM behaviour

- **No live pickup.** A `<service>` added to `sm.conf` after boot is unknown to SM until the next
  boot: info answers `STATUS_NOT_FOUND`, launch answers `107:Not Found`.
- **A commanded stop is final.** `smctl -s` on a service with `retry_count="3"` leaves it stopped.
  SM does not count it as a crash.
- **A killed process is restarted.** After `kill -9` of a `type="process"` service with
  `retry_count="3"` and `reset_board="no"`, SM starts it again, with no reboot.
  `smctl -g -t info` shows the counter as `<used>/<max>`.
- **An exit of a `reset_board="yes"` service reboots the unit.** When such a service exits (for
  example `Opera`, `retry_count="0"`), SM logs
  `configured as reset_board="yes" exited. Stop pinging watchdog` and
  `Expect a reboot in up to 5 seconds.`, writes a `saved_logs` archive and stops the watchdog. The
  board resets about 1 s later. The Opera case: [[01-architecture-and-boot]].

### Service environment

| Item | Value |
|---|---|
| stdout, stderr | `/tmp/redirlogs/stdout`, `/tmp/redirlogs/stderr` (FIFOs; do not `tail -f` them) |
| Open descriptors | inherited from SM: stdin `/dev/null`, `/dev/console`, 100+ sockets; not a clean fd table |
| `affinity_mask` | core pinning: `0x01` = core 0 (the `<services>` default), `0x02` = core 1 |
| User | without `user_account`, the service runs as `cmu`, which is uid 0 / gid 0 (`id`: `uid=0(cmu) gid=0(root)`) |
| `autorun="no"` | the service is listed but not started at boot |

syslog-ng reads both FIFOs as `pipe()` sources, program name `Std`: stdout at level `info`, stderr
at level `err`. All lines go to the rotating RAM log (`/var/log/running_log/`). Lines at `err` and
above also go to `/data_persist/log/error_logs/` on flash. A service's stderr therefore counts as
errors and is kept on flash. Details: [[Syslog-ng]].

## How to: run smctl from a shell

smctl needs the JCI libraries on the loader path:

```
LD_LIBRARY_PATH=/jci/lib:/usr/lib /jci/sm/smctl <options>
```

Without it, smctl fails with `libjcicommon_util.so: cannot open shared object`.

## Verified on

Firmware `cmu150_EU_74.00.324A`, bench unit.

## Open questions

- Whether `retry_count` counts per boot or over the lifetime of the service.
