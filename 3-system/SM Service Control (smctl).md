---
tags: [cmu150, sm, services, reference]
---

# SM service control (smctl)

The system manager `/jci/sm/sm` reads `/jci/sm/sm.conf` (or `sm_WCP.conf`) at boot and supervises the
listed `<service>` elements. `/jci/sm/smctl` is its command-line client.

## Running smctl from a shell

smctl needs the JCI libraries on the loader path:

```
LD_LIBRARY_PATH=/jci/lib:/usr/lib /jci/sm/smctl <options>
```

Without it: `libjcicommon_util.so: cannot open shared object`.

| Option | Meaning |
|---|---|
| `-l -w -n <svc>` | launch and wait |
| `-s -w -n <svc>` | stop and wait |
| `-r -n <svc>` | restart |
| `-g -t info -n <svc>` | print service info |

## Exit code is not a result

smctl exits 0 even when the command failed. The result is only in the text output:

- unknown service, info: `ERROR:smctl_get_service_info_command_cb [STATUS_NOT_FOUND] <svc>`
- unknown service, launch: `smctl_start_service_command_cb <svc> {107:Not Found}` and
  `WARNING:Starting service <svc> failed. Check if service with this name exists`
- known service, info: `[STATUS_SUCCESS] <svc> [6:RUNNING]` / `[9:STOPPED]`, followed by a
  `<service … state="RUNNING|STOPPED" … retry_count="<used>/<max>" pid="…"/>` element.

## Behaviour

- **No live pickup.** A `<service>` added to `sm.conf` after boot is unknown to SM until the next boot:
  info answers `STATUS_NOT_FOUND`, launch answers `107:Not Found`.
- **A commanded stop is final.** `smctl -s` on a service with `retry_count="3"` leaves it stopped; SM does not
  count it as a crash.
- **A killed process is restarted.** `kill -9` of a `type="process"` service with `retry_count="3"` and
  `reset_board="no"`: SM starts it again, no reboot. Whether `retry_count` counts per boot or over the
  service lifetime is not measured; `smctl -g -t info` shows the counter as `<used>/<max>`.
- **`reset_board="yes"` exit reboots the unit.** When such a service exits (e.g. `Opera`,
  `retry_count="0"`), SM logs `configured as reset_board="yes" exited. Stop pinging watchdog` and
  `Expect a reboot in up to 5 seconds.`, writes a `saved_logs` archive and stops the watchdog.
  The board resets about 1 s later. See jci-ui `01-architecture-and-boot` for the Opera case.
- Service stdout/stderr go to `/tmp/redirlogs/{stdout,stderr}` (FIFOs; do not `tail -f`).
  syslog-ng (`/etc/syslog-ng.conf`) reads both as `pipe()` sources, program name `Std`:
  stdout at level `info`, stderr at level `err`. All lines go to the rotating RAM log
  (`process-log /var/log/running_log/ 512`); `err` and above also pass `f_errors` +
  `f_useless` into `process-log /data_persist/log/error_logs/ 2048` on flash. A service's
  stderr therefore counts as errors and is kept on flash.
- A `type="process"` service inherits SM's open descriptors (stdin `/dev/null`, `/dev/console`,
  100+ sockets), not a clean fd table.
- `affinity_mask` pins a service to cores (`0x01` = core 0, the `<services>` default; `0x02` = core 1).
- A service without `user_account` runs as `cmu`, which is uid 0 / gid 0 (`id`: `uid=0(cmu) gid=0(root)`).
- `autorun="no"` services are listed but not started at boot.
