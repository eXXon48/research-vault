---
applet: "chattr"
tags: [busybox/command]
---

# `chattr`

```text
Usage: chattr [-R] [-+=AacDdijsStTu] [-v VERSION] [FILE]...

Change file attributes on an ext2 fs

Modifiers:
	-	Remove attributes
	+	Add attributes
	=	Set attributes
Attributes:
	A	Don't track atime
	a	Append mode only
	c	Enable compress
	D	Write dir contents synchronously
	d	Don't backup with dump
	i	Cannot be modified (immutable)
	j	Write all data to journal first
	s	Zero disk storage when deleted
	S	Write file contents synchronously
	t	Disable tail-merging of partial blocks with other files
	u	Allow file to be undeleted
	-R	Recurse
	-v	Set the file's version/generation number
```

---
[[BusyBox|← All BusyBox commands]]
