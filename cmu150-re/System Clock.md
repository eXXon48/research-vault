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
