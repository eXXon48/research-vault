---
applet: "arp"
tags: [busybox/command]
---

# `arp`

```text
Usage: arp 
[-vn]	[-H HWTYPE] [-i IF] -a [HOSTNAME]
[-v]		    [-i IF] -d HOSTNAME [pub]
[-v]	[-H HWTYPE] [-i IF] -s HOSTNAME HWADDR [temp]
[-v]	[-H HWTYPE] [-i IF] -s HOSTNAME HWADDR [netmask MASK] pub
[-v]	[-H HWTYPE] [-i IF] -Ds HOSTNAME IFACE [netmask MASK] pub

Manipulate ARP cache

	-a		Display (all) hosts
	-s		Set new ARP entry
	-d		Delete a specified entry
	-v		Verbose
	-n		Don't resolve names
	-i IF		Network interface
	-D		Read <hwaddr> from given device
	-A,-p AF	Protocol family
	-H HWTYPE	Hardware address type
```

---
[[BusyBox|← All BusyBox commands]]
