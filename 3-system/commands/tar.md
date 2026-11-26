---
applet: "tar"
tags: [busybox/command]
---

# `tar`

```text
Usage: tar -[cxtzjhmvO] [-X FILE] [-T FILE] [-f TARFILE] [-C DIR] [FILE]...

Create, extract, or list files from a tar file

Operation:
	c	Create
	x	Extract
	t	List
	f	Name of TARFILE ('-' for stdin/out)
	C	Change to DIR before operation
	v	Verbose
	z	(De)compress using gzip
	j	(De)compress using bzip2
	O	Extract to stdout
	h	Follow symlinks
	m	Don't restore mtime
	exclude	File to exclude
	X	File with names to exclude
	T	File with names to include
```

---
[[BusyBox|← All BusyBox commands]]
