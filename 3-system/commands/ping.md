---
applet: "ping"
tags: [busybox/command]
---

# `ping`

```text
Usage: ping [OPTIONS] HOST

Send ICMP ECHO_REQUEST packets to network hosts

	-4,-6		Force IP or IPv6 name resolution
	-c CNT		Send only CNT pings
	-s SIZE		Send SIZE data bytes in packets (default:56)
	-t TTL		Set TTL
	-I IFACE/IP	Use interface or IP address as source
	-W SEC		Seconds to wait for the first response (default:10)
			(after all -c CNT packets are sent)
	-w SEC		Seconds until ping exits (default:infinite)
			(can exit earlier with -c CNT)
	-q		Quiet, only displays output at start
			and when finished
```

---
[[BusyBox|← All BusyBox commands]]
