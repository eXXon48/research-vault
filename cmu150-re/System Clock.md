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
  every boot, before the 90 s mark, until the configs were restored to stock. No log of the
  failed boots survives (`/var/log/running_log` is tmpfs; its `log_000N.tar.gz` are size
  rotations of the current boot only). Likely mechanism: `/jci/sm/sm` waits with
  `pthread_cond_timedwait` and arms `timer_create`/`timer_settime` timeouts, and has no
  `pthread_condattr_setclock`, so its waits run on `CLOCK_REALTIME`. A forward jump of
  ~56 years expires every pending start/connect/ping timeout at once. The runtime jump
  above (all services already up) did not hit this. The clock must be set either before
  `/jci/sm/sm` starts or once all SM stages have finished starting.
- **Setting the clock once SM has finished starting works at boot.** The same SM `process`
  service before `stage_1`, but running a shell script that waits for
  `SM_SERVER_CMD_RUNNING` in `/tmp/smevents.txt` with no service still starting (plus a
  5 s settle) and only then runs `hwclock -s -u`, set the clock at about 64 s uptime. The
  boot completed normally: the service logs `SM_SVC_CMD_STARTED` at ~8.2 s (SM does not wait
  for the script), `sm` and `opera` kept their PIDs over the following minute, and no
  service was left starting. HMI time services follow the jump on their next 1-minute tick.
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
