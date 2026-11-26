---
applet: "traceroute"
tags: [busybox/command]
---

# `traceroute`

```text
Usage: traceroute [-FIldnrv] [-f 1ST_TTL] [-m MAXTTL] [-p PORT] [-q PROBES]
	[-s SRC_IP] [-t TOS] [-w WAIT_SEC] [-g GATEWAY] [-i IFACE]
	[-z PAUSE_MSEC] HOST [BYTES]

Trace the route to HOST

	-F	Set the don't fragment bit
	-I	Use ICMP ECHO instead of UDP datagrams
	-l	Display the TTL value of the returned packet
	-d	Set SO_DEBUG options to socket
	-n	Print numeric addresses
	-r	Bypass routing tables, send directly to HOST
	-v	Verbose
	-m	Max time-to-live (max number of hops)
	-p	Base UDP port number used in probes
		(default 33434)
	-q	Number of probes per TTL (default 3)
	-s	IP address to use as the source address
	-t	Type-of-service in probe packets (default 0)
	-w	Time in seconds to wait for a response (default 3)
	-g	Loose source route gateway (8 max)
```

---
[[BusyBox|← All BusyBox commands]]
