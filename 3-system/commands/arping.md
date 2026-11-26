---
applet: "arping"
tags: [busybox/command]
---

# `arping`

```text
Usage: arping [-fqbDUA] [-c CNT] [-w TIMEOUT] [-I IFACE] [-s SRC_IP] DST_IP

Send ARP requests/replies

	-f		Quit on first ARP reply
	-q		Quiet
	-b		Keep broadcasting, don't go unicast
	-D		Duplicated address detection mode
	-U		Unsolicited ARP mode, update your neighbors
	-A		ARP answer mode, update your neighbors
	-c N		Stop after sending N ARP requests
	-w TIMEOUT	Time to wait for ARP reply, seconds
	-I IFACE	Interface to use (default eth0)
	-s SRC_IP	Sender IP address
	DST_IP		Target IP address
```

---
[[BusyBox|← All BusyBox commands]]
