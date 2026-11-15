---
applet: "iproute"
tags: [busybox/command]
---

# `iproute`

```text
Usage: iproute { list | flush | { add | del | change | append |
		replace | monitor } ROUTE }

iproute { list | flush } SELECTOR
iproute get ADDRESS [from ADDRESS iif STRING]
			[oif STRING] [tos TOS]
iproute { add | del | change | append | replace | monitor } ROUTE
			SELECTOR := [root PREFIX] [match PREFIX] [proto RTPROTO]
			ROUTE := [TYPE] PREFIX [tos TOS] [proto RTPROTO]
				[metric METRIC]
```

---
[[BusyBox|← All BusyBox commands]]
