---
tags: [cmu150, time, hmi]
---

# System Clock

How the CMU150 keeps time: the unset Linux clock, the hardware RTC, the time service, why
setting the clock can break the boot, and the TLS trust store that depends on it.

## Overview

The Linux system clock is never set: after every boot it counts from 1970. The real time is
kept in the hardware RTC, in UTC, and nothing copies it into the system clock. The HMI shows
the correct time because the time service (BLM_TIME) reads the RTC and pushes the time into
the GUI.

The system clock can be set from the RTC (`hwclock -s -u`), but only after BLM_TIME has
started. BLM_TIME computes wait deadlines on the monotonic clock and passes them to calls that
measure the realtime clock. With a 1970 clock the two are nearly equal and the waits work.
With a real clock every wait times out at once. At start-up this crashes BLM_TIME in a loop
until the board resets. A preloaded library that corrects the deadlines removes the problem.

Related: [[Boot Timeline]], [[SM Service Control (smctl)]], [[05-framework-modules]]
(`framework.common` API).

## Reference

### Clocks on the unit

| Clock | Source | Value |
|---|---|---|
| Linux system clock (`CLOCK_REALTIME`) | not set at boot | counts from the epoch: `date` → `Thu Jan  1 00:31:41 UTC 1970` at ~1900 s uptime |
| Hardware RTC `/dev/rtc0` | `snvs_rtc` (i.MX SNVS) | real UTC in `/sys/class/rtc/rtc0/{date,time,since_epoch}` |
| HMI status-bar clock | pushed by the native side | correct local time (UTC+1 in the observed case) |

Anything that relies on wall time sees 1970: `find -mmin`, file mtimes, TLS certificate
validity, JS `new Date()` in the HMI page.

### RTC

- Driver module `rtc_snvs` (`/lib/modules/3.0.35/kernel/drivers/rtc/rtc-snvs.ko`), autoloaded
  for the platform device; no boot script loads it.
- dmesg: `snvs_rtc snvs_rtc.0: rtc core: registered snvs_rtc as rtc0` at 5.96 s.
- Read it directly from `/sys/class/rtc/rtc0/since_epoch`. No SM service is needed.
- No stock init script calls `hwclock` (the BusyBox applet `/sbin/hwclock`), so the RTC is
  never copied into the system clock.

### Setting the time in the HMI

- **Settings → Clock** changes the time of day only. There is no date field; the date comes
  from GPS.
- The change writes `/dev/rtc0` directly. Example: after setting 04:00 local, `since_epoch`
  moved from 10:59 to 03:04 UTC of the same day.
- So the user can move the RTC by up to about one day. The RTC is not a trusted time source:
  for TLS certificate validity, allow at least a day of skew.

### Time service (BLM_TIME)

| | |
|---|---|
| Binary | `/jci/time/svcjciblmtime.so` |
| D-Bus | `com.jci.BLM_TIME`: `GetClock`, `GetClockEx`, signal `ClockChanged(hours, mins, u32Timestamp, u64CallTimestamp)` |
| Source | `/dev/rtc0` plus timezone, DST and user offsets |
| GPS sync | through the LDS position service (`BLM_TIME_GPS_*`); needs a 2D/3D fix, reschedules when there is none; with GPS sync off the time is set manually in Settings |
| Sets the system clock | no: no `settimeofday`/`clock_settime` in its strings |

`/jci/time/svcjcitime.so` is the generic `com.jci.time` date/time API (`GetDatetime`,
`SetDatetimeComponents`, offset-change events).

### Time in the GUI

- The native side sends the time: `SendCurrentTimeEpoch` → `payload.u32TimestampSec` in
  `syssettingsApp.js`, `_timeChangedCallback`.
- The GUI pushes it to the status bar with `framework.common.updateSbClock(ms)`.
- `framework.common.getCurrentTime()` returns epoch milliseconds at minute granularity (seconds
  always `:00`), for example `1794580620000`.
- HMI time services follow a clock change on their next 1-minute tick.

### The clock mix-up in BLM_TIME

`BLM_TIME_SemTimedWait` (`blm_time_common.c`) computes the deadline as
`clock_gettime(CLOCK_MONOTONIC) + timeout` and passes it to `sem_timedwait`, which measures
against `CLOCK_REALTIME`.

| System clock | Effect |
|---|---|
| 1970 (stock) | the two clocks are nearly equal; waits work |
| real time | the deadline lies decades in the past; every wait fails at once with `ETIMEDOUT` |

The waits go through `BLM_TIME_wait_timedsync`. Its callers:

- `BLM_TIME2CPP_TIME_Init`
- `blm_time_getCppDatetime`
- `BLM_TIME2CPP_TIME_SetClock`, `…SetTimezone`, `…SetDaylightSaving`, `…SetDatetimeComponents`
- `CLOCK_SETTINGS_Initialize`, `CLOCK_SETTINGS_SetTimezone`, `CLOCK_SETTINGS_SetDST`

**If the clock is real before BLM_TIME starts:**

1. `BLM_TIME2CPP_TIME_Init` fails. Log: `timeout of timedwait occurred` →
   `timeout in connecting to CPP_Time occurred` → `Error in Initializing BLM_TIME`.
2. The service's `[SM]SvcStateChH` thread dies with `SIGSEGV` on the error path.
3. SM restarts the service about every 10 s.
4. After six crashes, `sm_health` runs `log-and-data-save.sh` and the board resets.

Evidence survives the reboot:

- cores in `/data_persist/log/dumps/`
  (`coreNNN-[SM]SvcStateChH-SIGSEGV(11)-<pid>-<epoch>-74.00.324A.core.bz2`);
- the running log of the failed boot in `/data_persist/log/saved_logs/logsNNN_coreNNN_*.tar.bz2`.

**If the clock is set after BLM_TIME has started:** nothing crashes, but every later timed wait
also times out at once. That covers clock, timezone and DST changes and GPS time sync. Example:
a timezone or time change in Settings → Clock is **not applied**.

- `blm_time_getCppDatetime` times out three times, then the log shows
  `Logic thread - max number of retry reached - operation will not be performed`.
- The RTC is left unchanged.
- During the retries BLM_TIME logs `service did NOT reset Logic Thread heart-beat`. There is no
  core, and SM restarts nothing.

A restart of BLM_TIME after the clock is set would repeat the crash loop.

### The same mix-up elsewhere

A decompile covered every JCI/vendor binary on the rootfs that calls an absolute-deadline wait
(117 binaries). It found three more places that take the deadline from `CLOCK_MONOTONIC` and pass
it to a realtime call. In glibc 2.11 these calls have no clock choice: `sem_timedwait`,
`pthread_mutex_timedlock`, `pthread_timedjoin_np`, `pthread_rwlock_timed*`.

| Binary | Function | Call |
|---|---|---|
| `/jci/lib/libbttr.so` | `bttr_prot_detach` | `sem_timedwait`, monotonic + 1 s |
| `/jci/pb/svcjcipb.so` (`jciPB`) | `pb_joinPbApplThread` | `pthread_timedjoin_np` |
| `/jci/sxms/svcjcisxms.so` (`jciSXMS`) | `OS_eSemTake` | `pthread_mutex_timedlock` |

On a real clock each of these returns `ETIMEDOUT` at once instead of waiting. The
`pthread_cond_timedwait` in `jciSXMS` uses a `CLOCK_MONOTONIC` condvar and is correct.

Everything else is consistent:

- realtime deadlines into realtime waits;
- monotonic deadlines into condvars created with `pthread_condattr_setclock(CLOCK_MONOTONIC)`;
- `TIMER_ABSTIME` timerfds on `CLOCK_MONOTONIC`;
- relative `timer_settime`.

No wait was found that takes a realtime deadline into a monotonic condvar. That case would hang
on a real clock.

### Fix: deadline-correcting preload library

A library listed in `/etc/ld.so.preload` defines the three affected calls (`sem_timedwait`,
`pthread_mutex_timedlock`, `pthread_timedjoin_np`). It moves a deadline onto the realtime clock
when both of these are true:

- the deadline is earlier than 2000;
- the realtime clock is past 2000.

The new deadline is `deadline + realtime_now − monotonic_now`. Each wait then lasts the intended
time. With the clock at 1970 the library changes nothing.

- The library must export only those symbols. A toolchain runtime linked into it (for example
  compiler-rt's weak `memcpy`/`memset`) would interpose libc in every process.
- A library listed in `/etc/ld.so.preload` stays mapped in every dynamically linked process
  that started with it (114 of 192 processes, including `init`). Deleting the library and
  `/etc/ld.so.preload` at runtime does not unmap it. After that, `mount -o remount,ro /` fails
  with "busy", and `/` stays read-write until the next reboot. The next boot starts without
  the library.

### What runs before SM

`/usr/bin/autostart` does the following:

1. Sources `/jci/scripts/start_normal_mode.sh`.
2. Sleeps 1 s.
3. Runs `get_board_type.sh`.
4. Only then starts `taskset 0x02 /jci/sm/sm -f <sm.conf|sm_WCP.conf> -e /tmp/smevents.txt &`.

Code appended to `start_normal_mode.sh` runs at about 6–7 s uptime. It can race the RTC
registration (5.96 s), so it has to wait for `/dev/rtc0`.

A static read of the jobs that `start_normal_mode.sh` starts in the background found nothing
that depends on the clock:

| Job | What it does |
|---|---|
| `redirect_core.sh`, `bt_drivers.sh`, `show_version.sh`, `set_hdcp_config.sh`, `set_nng_debug.sh` | one-shots: write `core_pattern`, `insmod`, read `version.ini`, write `/sys/class/nvram`, create a symlink; no time logic |
| `jci-log.sh start` | starts `syslog-ng` 3.3.5 (`libsyslog-ng-3.3.5.so`, bundled ivykis event loop with `clock_gettime`); only log timestamps follow the wall clock |
| `jci_cpu_gauge` | imports only `clock_gettime`, `gettimeofday`, `nanosleep`: relative sleeps, no absolute waits or timers |

The Logger (`/jci/jci … Logger`) and SM start after the script. Both long-running processes
also survived a clock change at runtime (see Measurements).

### Detecting that SM has finished starting

`/tmp/smevents.txt` (SM option `-e`) is a CSV `PID,Component,Event,Timestamp`. The timestamp is
seconds since boot, not wall time.

- Each service logs `SM_SVC_CMD_START` → (`SM_SVC_HANDSHAKE_DONE`, `SM_SVC_CMD_STARTING`) →
  `SM_SVC_CMD_STARTED` (or `SM_SVC_CMD_STOPPED`).
- SM itself logs `SM_SERVER_CMD_INIT`/`STARTING` at ~9.9 s, and `SM_SERVER_CMD_RUNNING` once
  the autorun set is up. On a bench unit: `RUNNING` at 58.9 s, last stage `stage_vr` at 51.8 s.
- A service can still start after `RUNNING` (`jciMSG` at 59.6 s). "All started" therefore
  means: `RUNNING` seen, and no service whose last event is `START`, `STARTING` or
  `HANDSHAKE_DONE`.
- SM `process` services run as `cmu`, which is uid 0.

### TLS and certificates

| | |
|---|---|
| Trust store | `/etc/ssl/certs/` holds only `cacert.pem`: the Mozilla bundle "as of Wed Apr 25 2012", 136 certificates, no modern roots such as ISRG Root X1 |
| libcurl | `libcurl.so.4.2.0` defaults to `/etc/ssl/certs/ca-certificates.crt`, which does not exist |
| OpenSSL | 1.0.1u (22 Sep 2016, TLS 1.2 max); `OPENSSLDIR` = `/usr/lib/ssl`, no `cert.pem` there |

Stock certificates in `/jci/certificates/` (and `dev/`):

| File | Validity |
|---|---|
| `jci_root_cert.pem` (CN `JohnsonControlsRoot`) | 2012-09-13 → 2032-09-13 |
| `certcrl.pem` (X.509 CRL) | `thisUpdate` 2013-04-12, `nextUpdate` 2013-04-20 |
| `dev/certcrl.pem` | 2013-01-31 → 2013-02-08 |

The CRL is outside its validity window on both clocks: not yet valid in 1970, expired since
2013 on a real clock. Its only stock reader is `/jci/lib/libjcisecurity.so` (used by
`/usr/bin/testCAAM`). Its OpenSSL imports are signature checks only (`X509_verify`,
`X509_CRL_verify`, `RSA_verify`, `PEM_read_X509_CRL`). It has no `X509_verify_cert`,
`X509_cmp_time` or `check_time` strings, so its checks do not depend on the system clock.

A static musl Rust client with rustls + ring (target `arm-unknown-linux-musleabi`) runs on
kernel 3.0.35. Randomness through `/dev/urandom` works; the kernel has no `getrandom` syscall.

## How to: set the system clock from the RTC

Without the preload library:

1. Add an SM `process` service before `stage_1` (one-shot, `exit_allowed="yes"`,
   `reset_board="no"`) that runs a shell script.
2. In the script, wait until SM has finished starting (see "Detecting that SM has finished
   starting"), then 5 s more.
3. Run `hwclock -s -u`.

SM does not wait for the script: the service logs `SM_SVC_CMD_STARTED` at ~8.2 s. After this,
Settings → Clock changes and GPS sync do not work (see "The clock mix-up in BLM_TIME").

With the preload library, `hwclock -s -u` can run before SM, at the end of
`start_normal_mode.sh` (after `/dev/rtc0` exists). Clock changes and GPS sync keep working.

The clock setting does not persist: every boot starts at 1970 again.

## Measurements

| Test | Result |
|---|---|
| `hwclock -s -u` at runtime, ~53 min uptime | no process exit or restart (same `ps` list after several minutes), no new errors; `mDNSResponder` logs `mDNSPlatformRawTime went backwards` and corrects itself; VWM maintenance reminders recalculate without a due flag (`SchdMaintDue 0`, `OilChangeDue 0`); status-bar clock, menus, media and phone normal |
| SM `process` service `/sbin/hwclock -s -u` before `stage_1` in `sm.conf`/`sm_WCP.conf` | no SM on every boot (no SSH, no HMI), before the 90 s mark, until the configs were restored; cause: BLM_TIME |
| Same service, script waits for SM to finish starting + 5 s | clock set at about 64 s uptime; boot normal; `sm` and `opera` kept their PIDs over the next minute; no service left starting |
| `hwclock -s -u` at the end of `start_normal_mode.sh`, no library | clock set at 7.9 s; board reset at about 75 s |
| Preload library under `qemu-arm` with stock glibc 2.11.1, real clock | monotonic-deadline waits went from 0 ms to the full 1000 ms; realtime-deadline waits unchanged |
| Preload library on the unit, clock set after SM start-up | boot normal; BLM_TIME, PB and SXMS map the library; a timezone change is applied: `blm_time_event (4-TZ chg …)`, `TIME_Client_SetDatetimeComposite … time is sync'd`, then `ClockChanged`; no `timeout of timedwait`, no `max number of retry reached` |
| Preload library, `hwclock -s -u` at the end of `start_normal_mode.sh` | clock set at 7.96 s; `SM_SERVER_CMD_RUNNING` at 59.5 s; BLM_TIME initialised and exchanged time with CPP_TIME; `sm`, `opera`, BLM_TIME, PB and SXMS kept their PIDs from 150 s to 250 s; no new core, no timed-wait errors |
| Preload library, manual time change in Settings → Clock | each hour step logs `blm_time_event (3-TS chg …)` and `BLM_TIME2CPP_TIME_SetClock Setting clock to <UTC>`; `since_epoch` follows; the system clock (`date`) does not change and picks up the new RTC time at the next boot's RTC-to-system copy |
| Preload library, GPS time sync switched on | logs `blm_time_getLocation_cb … Setting UTC to …` and `blm_time_event (7-new TIME …)`; RTC matched real UTC within 1 s |
| rustls TLS 1.3 handshake (ECDHE/ECDSA P-256) | about 40 ms on the i.MX6, peak RSS about 0.65 MB |

Recurring `BLM_CARPLAY … Invalid UTCtime`, DAB and Bluetooth errors appear on a bench unit with
and without a clock change (no GPS).

## Verified on

Firmware `cmu150_EU_74.00.324A`, legacy board.

## Open questions

- GPS time sync with a real clock and without the library is expected to fail like the manual
  change (same code path). Not observed: no GPS fix on the bench.
