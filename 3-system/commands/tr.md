---
applet: "tr"
tags: [busybox/command]
---

# `tr`

```text
Usage: tr [-cds] STRING1 [STRING2]

Translate, squeeze, or delete characters from stdin, writing to stdout

	-c	Take complement of STRING1
	-d	Delete input characters coded STRING1
	-s	Squeeze multiple output characters of STRING2 into one character
```

## Behavior (BusyBox v1.19.2)

- Escapes `'\n'`, `'\r'` and `'\0'` work: `tr '\n' '\0'` turns lines into NUL-separated records, and `tr -d '\r\0'` deletes CR and NUL.

---
[[BusyBox|← All BusyBox commands]]
