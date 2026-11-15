---
applet: "tcpsvd"
tags: [busybox/command]
---

# `tcpsvd`

```text
Usage: tcpsvd [-hEv] [-c N] [-C N[:MSG]] [-b N] [-u USER] [-l NAME] IP PORT PROG

Create TCP socket, bind to IP:PORT and listen
for incoming connection. Run PROG for each connection.

	IP		IP to listen on, 0 = all
	PORT		Port to listen on
	PROG ARGS	Program to run
	-l NAME		Local hostname (else looks up local hostname in DNS)
	-u USER[:GRP]	Change to user/group after bind
	-c N		Handle up to N connections simultaneously
	-b N		Allow a backlog of approximately N TCP SYNs
	-C N[:MSG]	Allow only up to N connections from the same IP
			New connections from this IP address are closed
			immediately. MSG is written to the peer before close
	-h		Look up peer's hostname
	-E		Don't set up environment variables
	-v		Verbose
```

---
[[BusyBox|← All BusyBox commands]]
