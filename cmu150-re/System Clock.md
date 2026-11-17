---
tags: [cmu150, time, hmi]
---

# System Clock

How time is kept on the CMU150. Firmware `74.00.324A`. Part of [[CMU150]].

- The Linux system clock (`CLOCK_REALTIME`) is **not set**: after boot it counts from the
  epoch (`date` → `Thu Jan  1 00:31:41 UTC 1970` at ~1900 s uptime). Anything relying on wall
  time (`find -mmin`, file mtimes, TLS certificate validity, JS `new Date()` in the HMI page)
  sees 1970.
- The HMI status-bar clock shows the correct local time. It is pushed into the GUI by the
  native side (`framework.common.updateSbClock(ms)`) and read back with
  `framework.common.getCurrentTime()` → epoch milliseconds at **minute granularity**
  (seconds always `:00`), e.g. `1794580620000`.
- **Hardware RTC holds real UTC.** `/dev/rtc0` = `snvs_rtc` (i.MX SNVS):
  `/sys/class/rtc/rtc0/{date,time,since_epoch}` hold UTC (the screen shows local time, UTC+1
  in the observed case). Nothing copies the RTC into the system clock at boot (no
  `hwclock -s`), hence the 1970 system time. The RTC value is readable directly from
  `/sys/class/rtc/rtc0/since_epoch`.
- **Manual clock setting writes the RTC.** Settings → Clock lets the user change the time of
  day only (no date field; the date comes from GPS). Changing it rewrites `/dev/rtc0`
  directly: after setting 04:00 local, `since_epoch` moved from 10:59 to 03:04 UTC of the same
  day. The RTC is therefore user-adjustable by up to about one day and is not a trusted time
  source (e.g. for TLS certificate validity, allow at least a day of skew).
- **Setting the system clock at runtime is tolerated.** `hwclock -s -u` at ~53 min uptime
  (system clock jumped from 1970 to the RTC's UTC) caused no process exit or restart (same
  `ps` list after several minutes) and no new errors. The only reaction: `mDNSResponder`
  logs `mDNSPlatformRawTime went backwards` and applies a correction factor itself. VWM
  maintenance reminders recalculate without raising a due flag (`SchdMaintDue 0`,
  `OilChangeDue 0`). HMI status-bar clock, menus, media and phone stay normal. The setting
  is volatile: the next boot is back at 1970. Recurring `BLM_CARPLAY … Invalid UTCtime`,
  DAB and Bluetooth errors are present before the jump too (no GPS on a bench unit).
- **Setting the clock while SM starts services breaks the boot.** An SM `process` service
  `/sbin/hwclock -s -u` placed before `stage_1` in `sm.conf`/`sm_WCP.conf` (one-shot,
  `exit_allowed="yes"`, `reset_board="no"`) left the unit without SM (no SSH, no HMI) on
  every boot, before the 90 s mark, until the configs were restored to stock. The cause is
  the BLM_TIME service, not SM's own timeouts (see "BLM_TIME cannot start on a real clock"
  below). The runtime jump above (BLM_TIME already initialised) did not hit this.
- **Setting the clock once SM has finished starting works at boot.** The same SM `process`
  service before `stage_1`, but running a shell script that waits for
  `SM_SERVER_CMD_RUNNING` in `/tmp/smevents.txt` with no service still starting (plus a
  5 s settle) and only then runs `hwclock -s -u`, set the clock at about 64 s uptime. The
  boot completed normally: the service logs `SM_SVC_CMD_STARTED` at ~8.2 s (SM does not wait
  for the script), `sm` and `opera` kept their PIDs over the following minute, and no
  service was left starting. HMI time services follow the jump on their next 1-minute tick.
- **Setting the clock before SM starts also breaks the boot.** `hwclock -s -u` run
  synchronously at the end of `start_normal_mode.sh` (after `/dev/rtc0` exists, before
  `autostart` starts `/jci/sm/sm`) set the clock at 7.9 s uptime. That boot reset itself at
  about 75 s. The cause is the same as above.
- **BLM_TIME cannot start on a real clock.** `/jci/time/svcjciblmtime.so`,
  `BLM_TIME_SemTimedWait` (`blm_time_common.c`), computes its deadline as
  `clock_gettime(CLOCK_MONOTONIC) + timeout` and passes it to `sem_timedwait`, which
  measures against `CLOCK_REALTIME`. With the clock at 1970 the two clocks are nearly equal
  and the wait works. With a real clock the deadline lies decades in the past, and every
  wait fails at once with `ETIMEDOUT`.
  - The waits go through `BLM_TIME_wait_timedsync`. Its callers are
    `BLM_TIME2CPP_TIME_Init`, `blm_time_getCppDatetime`, `BLM_TIME2CPP_TIME_SetClock`,
    `…SetTimezone`, `…SetDaylightSaving`, `…SetDatetimeComponents`,
    `CLOCK_SETTINGS_Initialize`, `CLOCK_SETTINGS_SetTimezone` and `CLOCK_SETTINGS_SetDST`.
  - At start-up `BLM_TIME2CPP_TIME_Init` fails. The log shows
    `timeout of timedwait occurred` → `timeout in connecting to CPP_Time occurred` →
    `Error in Initializing BLM_TIME`.
  - The service's `[SM]SvcStateChH` thread then dies with `SIGSEGV` on the error path.
  - SM restarts the service about every 10 s. After six crashes, `sm_health` runs
    `log-and-data-save.sh` and the board resets.
  - Evidence survives the reboot: cores in `/data_persist/log/dumps/`
    (`coreNNN-[SM]SvcStateChH-SIGSEGV(11)-<pid>-<epoch>-74.00.324A.core.bz2`) and the
    running log of the failed boot in `/data_persist/log/saved_logs/logsNNN_coreNNN_*.tar.bz2`.
  - Consequence: the clock may only be set **after** BLM_TIME has initialised. Once it is
    set, every later BLM_TIME timed wait (clock, timezone or DST change, GPS time sync) also
    times out at once. A restart of the BLM_TIME service after the jump would repeat the
    crash loop.
  - Observed after a post-SM jump: changing the timezone and the time in Settings → Clock
    is **not applied**. For each change, `blm_time_getCppDatetime` times out three times,
    then the log shows `Logic thread - max number of retry reached - operation will not be
    performed`. The RTC is left unchanged. BLM_TIME logs `service did NOT reset Logic
    Thread heart-beat` during the retries, but nothing crashes: no core, and SM restarts
    nothing. GPS time sync takes the same path, so it is expected to fail the same way
    (not observed: no GPS fix on the bench).
- **Other stock waits with the same clock mix-up.** A decompile of every JCI/vendor binary on
  the rootfs that calls an absolute-deadline wait (117 binaries) found three more places that
  take the deadline from `CLOCK_MONOTONIC` and pass it to a call that measures against
  `CLOCK_REALTIME` (in glibc 2.11 `sem_timedwait`, `pthread_mutex_timedlock`,
  `pthread_timedjoin_np` and `pthread_rwlock_timed*` have no clock choice):
  - `/jci/lib/libbttr.so` `bttr_prot_detach`: `sem_timedwait`, monotonic + 1 s.
  - `/jci/pb/svcjcipb.so` (`jciPB`) `pb_joinPbApplThread`: `pthread_timedjoin_np`.
  - `/jci/sxms/svcjcisxms.so` (`jciSXMS`) `OS_eSemTake`: `pthread_mutex_timedlock` (its
    `pthread_cond_timedwait` uses a `CLOCK_MONOTONIC` condvar and is correct).
  On a real clock each of these returns `ETIMEDOUT` at once instead of waiting. Everything
  else is consistent: realtime deadlines into realtime waits, monotonic deadlines into
  condvars created with `pthread_condattr_setclock(CLOCK_MONOTONIC)`, `TIMER_ABSTIME`
  timerfds on `CLOCK_MONOTONIC`, and relative `timer_settime`. No wait was found that takes a
  realtime deadline into a monotonic condvar (which would hang on a real clock).
- **The mix-up can be corrected without touching the binaries.** A preloaded library that
  defines the three calls, and moves a deadline earlier than 2000 onto the realtime clock
  (`deadline + realtime_now − monotonic_now`) when the realtime clock is past 2000, makes the
  pattern wait the intended time. Tested under `qemu-arm` with the stock glibc 2.11.1 on a
  real clock: monotonic-deadline waits went from 0 ms to the full 1000 ms; realtime-deadline
  waits are unchanged; with the clock at 1970 the library changes nothing. The library must
  export only those symbols: a toolchain runtime linked in (e.g. compiler-rt's weak
  `memcpy`/`memset`) would otherwise interpose libc in every process.
  - On the unit (library listed in `/etc/ld.so.preload`, clock set after SM start-up): the
    boot completes normally and BLM_TIME, PB and SXMS map the library. A timezone change
    in Settings → Clock is applied: BLM_TIME logs `blm_time_event (4-TZ chg …)`,
    `TIME_Client_SetDatetimeComposite … time is sync'd`, then broadcasts `ClockChanged`.
    It logs no `timeout of timedwait` and no `max number of retry reached`.
  - With the library preloaded, setting the clock **before SM starts** works:
    `hwclock -s -u` at the end of `start_normal_mode.sh` set the clock at 7.96 s uptime; SM
    reached `SM_SERVER_CMD_RUNNING` at 59.5 s, BLM_TIME initialised and exchanged time with
    CPP_TIME, `sm`/`opera`/BLM_TIME/PB/SXMS kept their PIDs from 150 s to 250 s, no new core
    in `/data_persist/log/dumps/`, no timed-wait errors. The same hook without the library
    reset the board at about 75 s.
  - A manual time change in Settings → Clock is applied the same way: each hour step logs
    `blm_time_event (3-TS chg …)` and `BLM_TIME2CPP_TIME_SetClock Setting clock to <UTC>`,
    and `/sys/class/rtc/rtc0/since_epoch` follows. The Linux system clock (`date`) is not
    changed by it; it only picks the new RTC time up at the next boot's RTC-to-system copy.
  - GPS time sync also works: switching it on logs `blm_time_getLocation_cb … Setting UTC to
    …` and `blm_time_event (7-new TIME …)`; the RTC then matched real UTC within 1 s.
  - Removing the library at runtime (deleting it and `/etc/ld.so.preload`) leaves it mapped
    in every dynamically linked process that started with it (114 of 192 processes,
    including `init`). `mount -o remount,ro /` then fails with "busy", so `/` stays
    read-write until the next reboot. The next boot starts without the library.
- **The RTC is available before SM.** `/dev/rtc0` is the i.MX6 SNVS RTC, driver module
  `rtc_snvs` (`/lib/modules/3.0.35/kernel/drivers/rtc/rtc-snvs.ko`). No boot script loads it
  explicitly (it is autoloaded for the platform device); dmesg shows
  `snvs_rtc snvs_rtc.0: rtc core: registered snvs_rtc as rtc0` at 5.96 s. `/usr/bin/autostart`
  sources `/jci/scripts/start_normal_mode.sh`, sleeps 1 s, runs `get_board_type.sh`, and only
  then starts `taskset 0x02 /jci/sm/sm -f <sm.conf|sm_WCP.conf> -e /tmp/smevents.txt &`. Code
  appended to `start_normal_mode.sh` runs at about 6–7 s uptime, so it can race the RTC
  registration and has to wait for `/dev/rtc0`. Reading the RTC needs no SM service.
- **What runs before SM is clock-insensitive (static read).** Of the jobs
  `start_normal_mode.sh` backgrounds, `redirect_core.sh`, `bt_drivers.sh`, `show_version.sh`,
  `set_hdcp_config.sh` and `set_nng_debug.sh` are one-shots (write `core_pattern`, `insmod`,
  read `version.ini`, write `/sys/class/nvram`, create a symlink) with no time logic.
  `jci-log.sh start` launches `syslog-ng` 3.3.5 (`libsyslog-ng-3.3.5.so`, bundled ivykis
  event loop with `clock_gettime`; only log timestamps follow the wall clock).
  `jci_cpu_gauge` imports only `clock_gettime`, `gettimeofday`, `nanosleep` (relative sleeps,
  no absolute waits or timers). The Logger (`/jci/jci … Logger`) and SM start after the script.
  Both long-lived processes also survived the runtime jump above.
- **"SM has finished starting" is observable.** `/tmp/smevents.txt` (SM `-e`) is a CSV
  `PID,Component,Event,Timestamp`; the timestamp is seconds since boot, not wall time. Each
  service logs `SM_SVC_CMD_START` → (`SM_SVC_HANDSHAKE_DONE`, `SM_SVC_CMD_STARTING`) →
  `SM_SVC_CMD_STARTED` (or `SM_SVC_CMD_STOPPED`). SM itself logs `SM_SERVER_CMD_INIT`/
  `STARTING` at ~9.9 s and `SM_SERVER_CMD_RUNNING` once the autorun set is up (58.9 s on a
  bench unit, last stage `stage_vr` at 51.8 s). One service (`jciMSG`) still started after
  `RUNNING` (59.6 s), so "all started" = `RUNNING` seen and no service whose last event is
  `START`/`STARTING`/`HANDSHAKE_DONE`. SM `process` services run as `cmu`, which is uid 0.
- **TLS on the unit:** a static musl Rust client with rustls + ring (target
  `arm-unknown-linux-musleabi`) runs on kernel 3.0.35; an ECDHE/ECDSA P-256 TLS 1.3
  handshake takes about 40 ms on the i.MX6, peak RSS about 0.65 MB. Randomness via
  `/dev/urandom` works (the kernel has no `getrandom` syscall).
- **Stock TLS trust store is from 2012 and partly missing.** `/etc/ssl/certs/` holds only
  `cacert.pem`: the Mozilla bundle "as of Wed Apr 25 2012", 136 certificates (no modern roots
  such as ISRG Root X1). `libcurl.so.4.2.0` defaults to `/etc/ssl/certs/ca-certificates.crt`,
  which does not exist. OpenSSL is 1.0.1u (22 Sep 2016, TLS 1.2 max), `OPENSSLDIR` =
  `/usr/lib/ssl` (no `cert.pem` there). `hwclock` is the busybox applet (`/sbin/hwclock`);
  no stock init script calls it.
- **Time service:** `/jci/time/svcjciblmtime.so` (dbus `com.jci.BLM_TIME`: `GetClock`,
  `GetClockEx`, signal `ClockChanged(hours, mins, u32Timestamp, u64CallTimestamp)`) keeps
  the wall clock from `/dev/rtc0` plus timezone / DST / user offsets, and syncs it from
  **GPS** via the LDS position service (`BLM_TIME_GPS_*`, needs a 2D/3D fix, reschedules
  on no fix; with GPS sync off the time is set manually in Settings). Its strings contain no
  `settimeofday`/`clock_settime`. `/jci/time/svcjcitime.so` is the generic `com.jci.time`
  date/time API (`GetDatetime`, `SetDatetimeComponents`, offset-change events).
- The GUI receives the time from the native side (`SendCurrentTimeEpoch` →
  `payload.u32TimestampSec` in `syssettingsApp.js`, `_timeChangedCallback`) and pushes it to
  the status bar with `updateSbClock`.
- Stock certificates in `/jci/certificates/` (and `dev/`): `jci_root_cert.pem`
  (CN `JohnsonControlsRoot`) valid 2012-09-13 → 2032-09-13; `certcrl.pem` is an X.509
  CRL: `thisUpdate` 2013-04-12, `nextUpdate` 2013-04-20 (`dev/certcrl.pem`: 2013-01-31 →
  2013-02-08). The CRL is outside its validity window for both a 1970 clock (not yet valid)
  and a real clock (expired since 2013).
- The only stock reader of `certcrl.pem` is `/jci/lib/libjcisecurity.so` (used by
  `/usr/bin/testCAAM`). Its OpenSSL imports are signature checks (`X509_verify`,
  `X509_CRL_verify`, `RSA_verify`, `PEM_read_X509_CRL`); it has no `X509_verify_cert` /
  `X509_cmp_time` / `check_time` strings, so its checks do not depend on the system clock.

See [[jci-ui/05-framework-modules]] for the `framework.common` API.
