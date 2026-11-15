---
applet: "brctl"
tags: [busybox/command]
---

# `brctl`

```text
Usage: brctl COMMAND [BRIDGE [INTERFACE]]

Manage ethernet bridges

Commands:
	show			Show a list of bridges
	addbr BRIDGE		Create BRIDGE
	delbr BRIDGE		Delete BRIDGE
	addif BRIDGE IFACE	Add IFACE to BRIDGE
	delif BRIDGE IFACE	Delete IFACE from BRIDGE
	setageing BRIDGE TIME		Set ageing time
	setfd BRIDGE TIME		Set bridge forward delay
	sethello BRIDGE TIME		Set hello time
	setmaxage BRIDGE TIME		Set max message age
	setpathcost BRIDGE COST		Set path cost
	setportprio BRIDGE PRIO		Set port priority
	setbridgeprio BRIDGE PRIO	Set bridge priority
	stp BRIDGE [1/yes/on|0/no/off]	STP on/off
```

---
[[BusyBox|← All BusyBox commands]]
