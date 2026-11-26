---
tags: [busybox, moc]
---

# BusyBox commands

Reference for the CMU's BusyBox, taken from the on-unit `--help` output.
Platform notes and reverse-engineering findings: [[README|CMU150 research]].

- **Build:** `BusyBox v1.19.2 (2020-02-14 18:04:06 JST) multi-call binary.`
- **Applets:** 274
- **Help text not captured** (applet blocks or acts on the system): 23
- **No help text** in this build: 8

> Each note holds the exact `--help` output compiled into *this* firmware, so it is authoritative for the unit, not a generic manual.


**Legend:** ⛔ help text not captured · ∅ no help text

## #
[[bracket-test|[]] · [[double-bracket-test|[[]]

## A
[[add-shell|add-shell]] · [[addgroup|addgroup]] · [[adduser|adduser]] · [[adjtimex|adjtimex]] · [[arp|arp]] · [[arping|arping]] · [[ash|ash]] ⛔ · [[awk|awk]]

## B
[[base64|base64]] · [[basename|basename]] · [[blkid|blkid]] · [[blockdev|blockdev]] · [[brctl|brctl]] · [[bunzip2|bunzip2]] · [[bzcat|bzcat]] · [[bzip2|bzip2]]

## C
[[cal|cal]] · [[cat|cat]] · [[catv|catv]] · [[chattr|chattr]] · [[chgrp|chgrp]] · [[chmod|chmod]] · [[chown|chown]] · [[chpasswd|chpasswd]] · [[chroot|chroot]] · [[chrt|chrt]] · [[chvt|chvt]] · [[cksum|cksum]] · [[clear|clear]] · [[cmp|cmp]] · [[comm|comm]] · [[cp|cp]] · [[cpio|cpio]] · [[cryptpw|cryptpw]] · [[cttyhack|cttyhack]] ⛔ · [[cut|cut]]

## D
[[date|date]] · [[dd|dd]] · [[deallocvt|deallocvt]] · [[delgroup|delgroup]] · [[deluser|deluser]] · [[depmod|depmod]] · [[devmem|devmem]] · [[df|df]] · [[dhcprelay|dhcprelay]] · [[diff|diff]] · [[dirname|dirname]] · [[dmesg|dmesg]] · [[dnsdomainname|dnsdomainname]] ∅ · [[dos2unix|dos2unix]] · [[du|du]] · [[dumpkmap|dumpkmap]] · [[dumpleases|dumpleases]]

## E
[[echo|echo]] · [[ed|ed]] · [[egrep|egrep]] ∅ · [[eject|eject]] · [[env|env]] · [[expand|expand]] · [[expr|expr]]

## F
[[false|false]] · [[fbset|fbset]] · [[fbsplash|fbsplash]] ⛔ · [[fdflush|fdflush]] · [[fdformat|fdformat]] · [[fdisk|fdisk]] · [[fgconsole|fgconsole]] · [[fgrep|fgrep]] ∅ · [[find|find]] · [[findfs|findfs]] · [[flock|flock]] · [[fold|fold]] · [[free|free]] · [[fsck|fsck]] · [[fsync|fsync]] · [[fuser|fuser]]

## G
[[getopt|getopt]] · [[getty|getty]] ⛔ · [[grep|grep]] · [[groups|groups]] · [[gunzip|gunzip]] · [[gzip|gzip]]

## H
[[halt|halt]] ⛔ · [[hd|hd]] · [[hdparm|hdparm]] · [[head|head]] · [[hexdump|hexdump]] · [[hostid|hostid]] · [[hostname|hostname]] · [[httpd|httpd]] ⛔ · [[hwclock|hwclock]]

## I
[[id|id]] · [[ifconfig|ifconfig]] · [[ifdown|ifdown]] · [[ifenslave|ifenslave]] · [[ifplugd|ifplugd]] · [[ifup|ifup]] · [[inetd|inetd]] ⛔ · [[insmod|insmod]] · [[install|install]] · [[ionice|ionice]] · [[iostat|iostat]] · [[ip|ip]] · [[ipaddr|ipaddr]] · [[ipcalc|ipcalc]] · [[ipcrm|ipcrm]] · [[ipcs|ipcs]] · [[iplink|iplink]] · [[iproute|iproute]] · [[iprule|iprule]] · [[iptunnel|iptunnel]]

## K
[[kbd_mode|kbd_mode]] · [[kill|kill]] · [[killall|killall]] · [[killall5|killall5]] · [[klogd|klogd]] ⛔

## L
[[last|last]] · [[less|less]] ⛔ · [[linux32|linux32]] ∅ · [[linux64|linux64]] ∅ · [[ln|ln]] · [[loadkmap|loadkmap]] · [[logger|logger]] · [[login|login]] ⛔ · [[logname|logname]] · [[logread|logread]] · [[losetup|losetup]] · [[ls|ls]] · [[lsattr|lsattr]] · [[lsmod|lsmod]] · [[lspci|lspci]] · [[lsusb|lsusb]] ∅ · [[lzop|lzop]] · [[lzopcat|lzopcat]]

## M
[[makedevs|makedevs]] · [[md5sum|md5sum]] · [[mdev|mdev]] · [[mkdir|mkdir]] · [[mke2fs|mke2fs]] · [[mkfifo|mkfifo]] · [[mkfs.ext2|mkfs.ext2]] · [[mknod|mknod]] · [[mkpasswd|mkpasswd]] · [[mkswap|mkswap]] · [[mktemp|mktemp]] · [[modinfo|modinfo]] · [[modprobe|modprobe]] · [[more|more]] ⛔ · [[mount|mount]] · [[mountpoint|mountpoint]] · [[mpstat|mpstat]] · [[mv|mv]]

## N
[[nameif|nameif]] · [[nbd-client|nbd-client]] · [[nc|nc]] ⛔ · [[netstat|netstat]] · [[nice|nice]] · [[nmeter|nmeter]] · [[nohup|nohup]] · [[nslookup|nslookup]] · [[ntpd|ntpd]] ⛔

## O
[[od|od]] · [[openvt|openvt]]

## P
[[passwd|passwd]] · [[patch|patch]] · [[pgrep|pgrep]] · [[pidof|pidof]] · [[ping|ping]] · [[pipe_progress|pipe_progress]] ∅ · [[pkill|pkill]] · [[pmap|pmap]] · [[poweroff|poweroff]] ⛔ · [[powertop|powertop]] · [[printenv|printenv]] · [[printf|printf]] · [[ps|ps]] · [[pscan|pscan]] · [[pstree|pstree]] · [[pwd|pwd]] · [[pwdx|pwdx]]

## R
[[rdate|rdate]] · [[rdev|rdev]] · [[readlink|readlink]] · [[readprofile|readprofile]] · [[realpath|realpath]] · [[reboot|reboot]] ⛔ · [[remove-shell|remove-shell]] · [[renice|renice]] · [[reset|reset]] · [[resize|resize]] · [[rev|rev]] · [[rm|rm]] · [[rmdir|rmdir]] · [[rmmod|rmmod]] · [[route|route]] · [[rpm|rpm]] · [[rpm2cpio|rpm2cpio]] · [[rtcwake|rtcwake]] · [[run-parts|run-parts]] · [[rx|rx]]

## S
[[script|script]] · [[scriptreplay|scriptreplay]] · [[sed|sed]] · [[seq|seq]] · [[setarch|setarch]] · [[setconsole|setconsole]] ⛔ · [[setfont|setfont]] · [[setkeycodes|setkeycodes]] · [[setlogcons|setlogcons]] · [[setsid|setsid]] · [[sh|sh]] ⛔ · [[showkey|showkey]] · [[slattach|slattach]] · [[sleep|sleep]] · [[smemcap|smemcap]] · [[sort|sort]] · [[split|split]] · [[stat|stat]] · [[strings|strings]] · [[stty|stty]] · [[su|su]] · [[sulogin|sulogin]] ⛔ · [[sum|sum]] · [[swapoff|swapoff]] · [[swapon|swapon]] · [[sync|sync]] · [[sysctl|sysctl]] · [[syslogd|syslogd]] ⛔

## T
[[tac|tac]] · [[tail|tail]] · [[tar|tar]] · [[taskset|taskset]] · [[tcpsvd|tcpsvd]] · [[tee|tee]] · [[test|test]] ∅ · [[time|time]] · [[timeout|timeout]] · [[top|top]] ⛔ · [[touch|touch]] · [[tr|tr]] · [[traceroute|traceroute]] · [[true|true]] · [[tty|tty]] · [[ttysize|ttysize]] · [[tunctl|tunctl]] · [[tune2fs|tune2fs]]

## U
[[udhcpc|udhcpc]] ⛔ · [[udhcpd|udhcpd]] ⛔ · [[umount|umount]] · [[uname|uname]] · [[unexpand|unexpand]] · [[uniq|uniq]] · [[unix2dos|unix2dos]] · [[unlzop|unlzop]] · [[uptime|uptime]] · [[users|users]] · [[usleep|usleep]]

## V
[[vconfig|vconfig]] · [[vi|vi]] ⛔ · [[vlock|vlock]] · [[volname|volname]]

## W
[[watch|watch]] · [[watchdog|watchdog]] · [[wc|wc]] · [[wget|wget]] · [[which|which]] · [[who|who]] · [[whoami|whoami]]

## X
[[xargs|xargs]]

## Y
[[yes|yes]]

## Z
[[zcat|zcat]] · [[zcip|zcip]]

---
## Full applet list (as the unit prints it)

```text
BusyBox v1.19.2 (2020-02-14 18:04:06 JST) multi-call binary.
Copyright (C) 1998-2011 Erik Andersen, Rob Landley, Denys Vlasenko
and others. Licensed under GPLv2.
See source distribution for full notice.

Usage: busybox [function] [arguments]...
   or: busybox --list[-full]
   or: function [arguments]...

	BusyBox is a multi-call binary that combines many common Unix
	utilities into a single executable.  Most people will create a
	link to busybox for each function they wish to use and BusyBox
	will act like whatever it was invoked as.

Currently defined functions:
	[, [[, add-shell, addgroup, adduser, adjtimex, arp, arping, ash, awk,
	base64, basename, blkid, blockdev, brctl, bunzip2, bzcat, bzip2, cal,
	cat, catv, chattr, chgrp, chmod, chown, chpasswd, chroot, chrt, chvt,
	cksum, clear, cmp, comm, cp, cpio, cryptpw, cttyhack, cut, date, dd,
	deallocvt, delgroup, deluser, depmod, devmem, df, dhcprelay, diff,
	dirname, dmesg, dnsdomainname, dos2unix, du, dumpkmap, dumpleases,
	echo, ed, egrep, eject, env, expand, expr, false, fbset, fbsplash,
	fdflush, fdformat, fdisk, fgconsole, fgrep, find, findfs, flock, fold,
	free, fsck, fsync, fuser, getopt, getty, grep, groups, gunzip, gzip,
	halt, hd, hdparm, head, hexdump, hostid, hostname, httpd, hwclock, id,
	ifconfig, ifdown, ifenslave, ifplugd, ifup, inetd, insmod, install,
	ionice, iostat, ip, ipaddr, ipcalc, ipcrm, ipcs, iplink, iproute,
	iprule, iptunnel, kbd_mode, kill, killall, killall5, klogd, last, less,
	linux32, linux64, ln, loadkmap, logger, login, logname, logread,
	losetup, ls, lsattr, lsmod, lspci, lsusb, lzop, lzopcat, makedevs,
	md5sum, mdev, mkdir, mke2fs, mkfifo, mkfs.ext2, mknod, mkpasswd,
	mkswap, mktemp, modinfo, modprobe, more, mount, mountpoint, mpstat, mv,
	nameif, nbd-client, nc, netstat, nice, nmeter, nohup, nslookup, ntpd,
	od, openvt, passwd, patch, pgrep, pidof, ping, pipe_progress, pkill,
	pmap, poweroff, powertop, printenv, printf, ps, pscan, pstree, pwd,
	pwdx, rdate, rdev, readlink, readprofile, realpath, reboot,
	remove-shell, renice, reset, resize, rev, rm, rmdir, rmmod, route, rpm,
	rpm2cpio, rtcwake, run-parts, rx, script, scriptreplay, sed, seq,
	setarch, setconsole, setfont, setkeycodes, setlogcons, setsid, sh,
	showkey, slattach, sleep, smemcap, sort, split, stat, strings, stty,
	su, sulogin, sum, swapoff, swapon, sync, sysctl, syslogd, tac, tail,
	tar, taskset, tcpsvd, tee, test, time, timeout, top, touch, tr,
	traceroute, true, tty, ttysize, tunctl, tune2fs, udhcpc, udhcpd,
	umount, uname, unexpand, uniq, unix2dos, unlzop, uptime, users, usleep,
	vconfig, vi, vlock, volname, watch, watchdog, wc, wget, which, who,
	whoami, xargs, yes, zcat, zcip
```
