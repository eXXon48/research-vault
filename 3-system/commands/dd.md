---
applet: "dd"
tags: [busybox/command]
---

# `dd`

```text
Usage: dd [if=FILE] [of=FILE] [ibs=N] [obs=N] [bs=N] [count=N] [skip=N]
	[seek=N] [conv=notrunc|noerror|sync|fsync]

Copy a file with converting and formatting

	if=FILE		Read from FILE instead of stdin
	of=FILE		Write to FILE instead of stdout
	bs=N		Read and write N bytes at a time
	ibs=N		Read N bytes at a time
	obs=N		Write N bytes at a time
	count=N		Copy only N input blocks
	skip=N		Skip N input blocks
	seek=N		Skip N output blocks
	conv=notrunc	Don't truncate output file
	conv=noerror	Continue after read errors
	conv=sync	Pad blocks with zeros
	conv=fsync	Physically write data out before finishing

Numbers may be suffixed by c (x1), w (x2), b (x512), kD (x1000), k (x1024),
MD (x1000000), M (x1048576), GD (x1000000000) or G (x1073741824)
```

---
[[BusyBox|← All BusyBox commands]]
