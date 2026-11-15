---
applet: "vconfig"
tags: [busybox/command]
---

# `vconfig`

```text
Usage: vconfig COMMAND [OPTIONS]

Create and remove virtual ethernet devices

	add		[interface-name] [vlan_id]
	rem		[vlan-name]
	set_flag	[interface-name] [flag-num] [0 | 1]
	set_egress_map	[vlan-name] [skb_priority] [vlan_qos]
	set_ingress_map	[vlan-name] [skb_priority] [vlan_qos]
	set_name_type	[name-type]
```

---
[[BusyBox|← All BusyBox commands]]
