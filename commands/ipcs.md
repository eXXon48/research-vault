---
applet: "ipcs"
tags: [busybox/command]
---

# `ipcs`

```text
Usage: ipcs [[-smq] -i shmid] | [[-asmq] [-tcplu]]

	-i	Show specific resource
Resource specification:
	-m	Shared memory segments
	-q	Message queues
	-s	Semaphore arrays
	-a	All (default)
Output format:
	-t	Time
	-c	Creator
	-p	Pid
	-l	Limits
	-u	Summary
```

---
[[BusyBox|← All BusyBox commands]]
