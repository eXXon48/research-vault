---
applet: "ipaddr"
tags: [busybox/command]
---

# `ipaddr`

```text
Usage: ipaddr { {add|del} IFADDR dev STRING | {show|flush}
		[dev STRING] [to PREFIX] }

ipaddr {add|delete} IFADDR dev STRING
ipaddr {show|flush} [dev STRING] [scope SCOPE-ID]
	[to PREFIX] [label PATTERN]
	IFADDR := PREFIX | ADDR peer PREFIX
	[broadcast ADDR] [anycast ADDR]
	[label STRING] [scope SCOPE-ID]
	SCOPE-ID := [host | link | global | NUMBER]
```

---
[[BusyBox|← All BusyBox commands]]
