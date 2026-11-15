---
applet: "ifplugd"
tags: [busybox/command]
---

# `ifplugd`

```text
Usage: ifplugd [OPTIONS]

Network interface plug detection daemon

	-n		Don't daemonize
	-s		Don't log to syslog
	-i IFACE	Interface
	-f/-F		Treat link detection error as link down/link up
			(otherwise exit on error)
	-a		Don't up interface at each link probe
	-M		Monitor creation/destruction of interface
			(otherwise it must exist)
	-r PROG		Script to run
	-x ARG		Extra argument for script
	-I		Don't exit on nonzero exit code from script
	-p		Don't run script on daemon startup
	-q		Don't run script on daemon quit
	-l		Run script on startup even if no cable is detected
	-t SECS		Poll time in seconds
	-u SECS		Delay before running script after link up
	-d SECS		Delay after link down
	-m MODE		API mode (mii, priv, ethtool, wlan, iff, auto)
	-k		Kill running daemon
```

---
[[BusyBox|← All BusyBox commands]]
