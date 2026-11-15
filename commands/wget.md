---
applet: "wget"
tags: [busybox/command]
---

# `wget`

```text
Usage: wget [-c|--continue] [-s|--spider] [-q|--quiet] [-O|--output-document FILE]
	[--header 'header: value'] [-Y|--proxy on/off] [-P DIR]
	[--no-check-certificate] [-U|--user-agent AGENT] [-T SEC] URL...

Retrieve files via HTTP or FTP

	-s	Spider mode - only check file existence
	-c	Continue retrieval of aborted transfer
	-q	Quiet
	-P DIR	Save to DIR (default .)
	-T SEC	Network read timeout is SEC seconds
	-O FILE	Save to FILE ('-' for stdout)
	-U STR	Use STR for User-Agent header
	-Y	Use proxy ('on' or 'off')
```

## On the CMU150

- The unit reaches hosts on its LAN over plain HTTP: `wget -q -O - http://<host>:<port>/<file>`
  returns the file (outbound TCP is not filtered on the unit side).
- The `curl` on the unit is built without HTTPS: `curl: (1) Protocol https not supported or disabled in libcurl`.
- The system clock is 1970 after every boot ([[System Clock]]), so TLS certificate validity checks
  against it fail regardless of transport.

---
[[BusyBox|← All BusyBox commands]]
