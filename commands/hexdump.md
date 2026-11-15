---
applet: "hexdump"
tags: [busybox/command]
---

# `hexdump`

```text
Usage: hexdump [-bcCdefnosvxR] [FILE]...

Display FILEs (or stdin) in a user specified format

	-b		One-byte octal display
	-c		One-byte character display
	-C		Canonical hex+ASCII, 16 bytes per line
	-d		Two-byte decimal display
	-e FORMAT_STRING
	-f FORMAT_FILE
	-n LENGTH	Interpret only LENGTH bytes of input
	-o		Two-byte octal display
	-s OFFSET	Skip OFFSET bytes
	-v		Display all input data
	-x		Two-byte hexadecimal display
	-R		Reverse of 'hexdump -Cv'
```

---
[[BusyBox|← All BusyBox commands]]
