---
applet: "du"
tags: [busybox/command]
---

# `du`

```text
Usage: du [-aHLdclsxhmk] [FILE]...

Summarize disk space used for each FILE and/or directory.
Disk space is printed in units of 1024 bytes.

	-a	Show file sizes too
	-L	Follow all symlinks
	-H	Follow symlinks on command line
	-d N	Limit output to directories (and files with -a) of depth < N
	-c	Show grand total
	-l	Count sizes many times if hard linked
	-s	Display only a total for each argument
	-x	Skip directories on different filesystems
	-h	Sizes in human readable format (e.g., 1K 243M 2G )
	-m	Sizes in megabytes
	-k	Sizes in kilobytes (default)
```

---
[[BusyBox|← All BusyBox commands]]
