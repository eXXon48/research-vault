---
applet: "slattach"
tags: [busybox/command]
---

# `slattach`

```text
Usage: slattach [-cehmLF] [-s SPEED] [-p PROTOCOL] DEVICE

Attach network interface(s) to serial line(s)

	-p PROT	Set protocol (slip, cslip, slip6, clisp6 or adaptive)
	-s SPD	Set line speed
	-e	Exit after initializing device
	-h	Exit when the carrier is lost
	-c PROG	Run PROG when the line is hung up
	-m	Do NOT initialize the line in raw 8 bits mode
	-L	Enable 3-wire operation
	-F	Disable RTS/CTS flow control
```

---
[[BusyBox|← All BusyBox commands]]
