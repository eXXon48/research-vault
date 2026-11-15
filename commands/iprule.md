---
applet: "iprule"
tags: [busybox/command]
---

# `iprule`

```text
Usage: iprule {[list | add | del] RULE}

iprule [list | add | del] SELECTOR ACTION
	SELECTOR := [from PREFIX] [to PREFIX] [tos TOS] [fwmark FWMARK]
			[dev STRING] [pref NUMBER]
	ACTION := [table TABLE_ID] [nat ADDRESS]
			[prohibit | reject | unreachable]
			[realms [SRCREALM/]DSTREALM]
	TABLE_ID := [local | main | default | NUMBER]
```

---
[[BusyBox|← All BusyBox commands]]
