---
applet: "comm"
tags: [busybox/command]
---

# `comm`

```text
Usage: comm [-123] FILE1 FILE2

Compare FILE1 with FILE2

	-1	Suppress lines unique to FILE1
	-2	Suppress lines unique to FILE2
	-3	Suppress lines common to both files
```

## Behavior (BusyBox v1.19.2)

- `-` as FILE1 or FILE2 reads stdin (`... | comm -23 - FILE` works).

---
[[BusyBox|← All BusyBox commands]]
