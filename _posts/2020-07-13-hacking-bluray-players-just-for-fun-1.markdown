---
layout: post
slug: hacking-bluray-players-just-for-fun-1
title:  "Hacking BluRay players just for fun - part 1"
date:   2020-07-13 01:00:00
categories:
- infosec
tags:
- java
- bluray
- embedded
- hacking
---


My family has been complaining whenever we watch a DVD movie, that the screen kept glitching. To solve this problem I bought a new BluRay player. The players come with an Ethernet port these days, so I decided to have a look around before deplying it to the loungeroom...

## Editorial?

Movies for many people these days is via Fetch, or NetFlix (which we have) or $STREAMING_SERVICE which is convenient. So why I am I replacing the BluRay player? We have a collection of movies on disc which we like to rewatch, also our local general store still rents DVDs for $5 and I kind of like popping in for a chat and grabbing a movie to hire, its a bit old skool but still fun!

Anyway I digress. Just for fun, I'm going to apply a some cybersecurity buzzwords (I mean domain knowledge) to practice a some topics I've been revising, in order to further consolidate my knowledge.

## Planning I mean Unboxing

I purchased a new, and really quite cheap for what you get (~ $130, can probably be had for less with haggling) non-homebrand BluRay player from a certain Aussie-wide vendor that I don't avoid, unlike hardly normal. An interesting observation is the manufacturer provides firmware downloads on their webside, and the device can be upgraded from a USB stick.  This means I was able to download the firmware while the gadget was in the mail, and unpack it with BinWalk for a bit of a look-see. More on that later.

Specs:
- the player is smaller physically than it looks from online images
- 1x IR remote, front USB port, rear Ethernet port, read HDMI
- the 1x HDMI is the only output. I guess thats how this device can be so much smaller, my previous player had Component and yellow/red/white A/V outputs as well as HDMI
- power consumption is standby 0.5 watt - can't be avoided I guess.  As a bonus the power brick can be unplugged from the device, which is a nice touch
- the power brick itself however, is "sideways" which is really annoying - this means you cant have another power plug next to it both sides on a powerboard on one side without using a vertical double adaptor. ugh.
- the only buttons are Eject, and On/Off.
- allegedly it will play a laundry list of formats
- it would presumably be Region 4 - although there may be workarounds
- I didn't bother paying a couple $100 extra for the wifi version
- There is a 'Java Powered' logo on the back

TBH originally I wasn't planning to hook it up to the net anyway for normal use, connected to our TV I already have an old laptop setup for NetFlix which comes in handy for other purposes, and is more amenable to security hardening. But I might consider it later, depending on what I find.

The objective of this exercise is to see if there are anything interesting aspects to the player that could be modified to use the ethernet port, besides watch a movie with the family. In terms of business objectives, familiy use outweighs everything else, so I might be under pressure to leave it alone, and dont dare break the thing...

## Discovery - Research

I downloaded the firmware for the player from the manufacturer website.

What am I expecting to find?
- Hopefully a Linux operating system which means I could try and build and install additional toys - if I get time, which is not super likely sadly
- Java apps - as above

I did run it through binwalk, but that was a couple of weeks before I actually purchased the device and it finally rocked up. Instead, lets get into the nuts and bolts as though I'd never taken a look.

## Discovery - Reconaissance

Rather than using the player like a normal person, and then seeing what else I could do, I've never even turned it on yet and I'm instead connecting it to my test laboratory. This exercise is for fun after all!

Hypotheses:
- will it auto ask for DHCP on power up, before I've even turned it on connected to the Tele and configured it? Lets see
- will it have a static IP and if so can we work that out by what it spits out if any?

Note, I'm haven't read any manuals yet other than the quickstart guide for the specs, I probably should, but for fun I'm going in blind.

```
 +-------------------+               +-----------------+
 |                   |               |                 |
 |                   |               |                 |
 |   Intel NUC   eth1+---------------+  BluRay player  |
 |                   |               |                 |
 |   Debian 10       |               |                 |
 |                   |               |                 |
 |       eth0        |               |                 |
 +-------------------+               +-----------------+
          |
          |
          v
       my network
```

(diagram created with [asciiflow](http://asciiflow.com/)

First task - log everything output on powerup.

My lab includes an Intel NUC running debian I use for experiments like this. I I set up the NUC port (eth1 in the diagram, in practice something like enx00e04cabcdef as it is a USB adaptor) with a static IP, and set it to stay up even if the link is down so that tcpdump wont be interrupted.  To keep it simple I also disable ipv6:

```
nmcli c add con-name "USB-ether-1" ifname enx00e04cabcdef type ethernet \
  ip4 10.5.5.1 ip4 192.168.0.253
nmcli c mod "USB-ether-1" ipv6.method ignore
nmcli c up "USB-ether-1"
```

Run tcpdump and lets see what we can get. This command saves to a pcap file and also dumps to the console:

```
tcpdump  -n -i enx00e04cabcdef -U -s 0 -w - | tee capture.pcap | tcpdump -r - -e -l -n
```

What this does:
- `-n` avoid DNS lookups
- `-U` send data through on each packet instead of until the buffer is full
- `-s 0` capture entire packets, so we can se them in Wireshark later if desired
- `-w -` send pcap to standard out
- `-e` dump link headers as well
- `-r -` read packets from stdin
- `-l` line buffer the output, usually needed when using tcpdump to avoid stuttering and make CTRL+C work as expected

Power on, and what happened?

- the NUC started transmitting ipv6 router solicitation packets (annoying) - I should do something about that
- after a few seconds I saw packets from the player - bootp requests (✅)
- after a minute, it gave up on bootp and switched to APIPA, and output ARP packets to check 169.254.16.84 was OK to use
- then it output a couple of IGMPv3 packets
- after a couple more minutes it again went into a bootp loop

Not completely unexpected.

In the meantime I modified my connection to see if I can ping the device on the APIPA address

```
nmcli c mod "USB-ether-1" +ipv4.addresses "169.254.33.33/16"
nmcli d reapply enx00e04cabcdef
ping 169.254.16.84
```

That worked! Time to fire up nmap.

Note, all this time those bootp are still being recorded, as is the ping, so that will serve as a marker. The nmap scanning will also end up in the pcap file, so lots to sort through later.

Start by doing a quick TCP port scan.
```
nmap -p1-65535 -v -sT 169.254.16.84
```

Bingo:
```
Not shown: 65532 closed ports
PORT      STATE SERVICE
9222/tcp  open  teamcoherence
51623/tcp open  unknown
56789/tcp open  unknown
```

Well, there are some ports open, not sure what they are, although 9222 rings a bell.

Lets try and guess the O/S:
```
nmap -A -p9222,51623,56789 -v -sT 169.254.16.84
```

This took its time. So whatever is listening is not handling the guess packets very well. A few minutes later:

```
PORT      STATE SERVICE VERSION
9222/tcp  open  http    Google Chromecast httpd
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Content shell remote debugging
51623/tcp open  http    Mongoose httpd
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-title: Site doesn't have a title (text/plain).
56789/tcp open  http    Mongoose httpd
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-title: Site doesn't have a title (text/plain).
MAC Address: B4:6C:47:08:D2:EC (Unknown)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Ruckus ZoneFlex R710 WAP (Linux 3.4) (98%), Linux 2.6.32 - 3.10 (98%), Linux 3.2 - 3.16 (96%), Linux 3.2 - 4.9 (96%), Linux 3.4 - 3.10 (96%), Synology DiskStation Manager 5.2-5644 (96%), Linux 2.6.32 - 3.5 (95%), Linux 2.6.32 - 3.13 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 2.6.32 (95%)
No exact OS matches for host (test conditions non-ideal).
```

Chromecast should be obvious I guest, this is a network BluRay player. Actually, quite handy - means I can possibly use this player to cast my iPad and Galaxy Tab screens, so I have a reason to end up connecting this to my network (even if not the Internet)

We can look at those web services later.

Linux makes sense. We can confirm that when unpacking the firmware upgrade file with luck. (Spoiler: the firmware was a Linux filesystem image)

At this stage I noticed the tcpdump stopped, something seems to have crashed after that scan... cant ping anymore either. Power cycle time.

After the reboot it got the same APIPA address, good to know.

Lets do a quick UDP scan. Not really quick, because it takes a few seconds _each_ for all 65535 ports to repond with unreachable (or timeout) - tcpdump is showung unreachables, so there is no firewall, at least, not one configured to drop. (Not that I expected a firewall on a BluRay player, but perhaps after this it might be something I can add. Not that I expect to have the time, but it might be feasible...)

```
nmap -p1-65535 -v -sU -Pn 169.254.16.84
```

Eventually, after 20 minutes or so, the output stopped again, theoretically nmap was suggesting with 8 hours to do.

I now think this is a sleep mode. So it probably wont be feasible to do a full UDP scan without otherwise keeping the device up. Not critical, we are not searching for an RTSP feed or anything.

We can still run a quick UDP (1000 common port) scan:

```
nmap  -v -sU -Pn 169.254.16.84
```

After that, no open ports.

It is also possible additional TCP or UDP ports might open up once the device gets a DHCP address or is otherwise configured. Might be worth scanning again later.

Later I tried to discover the HTTP services.

This time around, 51623 was not open, so perhaps thats dynamically allocated? Need to run a syn scan again. THis time it was on 51376

Attempting to use curl to 56789 returned a 404. Same for 51376. So too little to go on at this stage as to what it might be.

Curl to 9222 returned a HTML web page, this is apparently a Chromecast debugging session. I'll need to do some research on that.

## Discovery - Research (again)

So, I downloaded the firmware, the image is approx 125 MByte.

If I wanted to flash it I would have to
- Decompress the download and copy it to the root of a FAT or NTFS USB stick
- Turn on with the USB stick in, if correctly named the update should happen automatically
- wait 5 minutes, don't lose power (or do this on a UPS)

The firmware is downloaded as a self extracting Windows exe. On Linux you can use unrar to unpack it.

Then I ran the unpacked file through binwalk:

```
binwalk redacted.FRM
```

I'm probably being paranoid here but I've redacted the manufacturer. SHould not be hard to work this out for yourself with some digging though. And Isuspect many vendors are similar in design.

Found all sorts of interesting things:

```
66650         0x1045A         Mediatek bootloader
117275        0x1CA1B         Mediatek bootloader
128540        0x1F61C         Boot section Start 0x5E5E5E5E End 0x20620
139317        0x22035         Certificate in DER format (x509 v3), header length: 4, sequence length: 1280
416037        0x65925         Certificate in DER format (x509 v3), header length: 4, sequence length: 1284
416153        0x65999         Certificate in DER format (x509 v3), header length: 4, sequence length: 1288
421760        0x66F80         CRC32 polynomial table, little endian
436176        0x6A7D0         CRC32 polynomial table, little endian
1121778       0x111DF2        mcrypt 2.2 encrypted data, algorithm: DES, mode: ECB, keymode: 8bit
1583040       0x1827C0        uImage header, header size: 64 bytes, header CRC: 0x5B7DCEF6, created: 2018-05-18 03:25:22, image size: 2297336 bytes, Data Address: 0x4000000, Entry Point: 0x4000000, data CRC: 0xD3733800, OS: Linux, CPU: ARM, image type: OS Kernel Image, compression type: none, image name: ""
1583104       0x182800        Linux kernel ARM boot executable zImage (little-endian)
1601044       0x186E14        gzip compressed data, maximum compression, from Unix, last modified: 1970-01-01 00:00:00 (null date)
16447232      0xFAF700        Squashfs filesystem, little endian, version 4.0, compression:gzip, size: 111831860 bytes, 2380 inodes, blocksize: 131072 bytes, created: 2018-05-18 06:27:41
128411392     0x7A76700       PNG image, 1920 x 1080, 8-bit/color RGBA, non-interlaced
```

This means, depending on whether / how checksums are used and the like, I could theoretically install additional binaries into the image, change what I guess might be the boot image (PNG), and so on. I guess the x509 certificates might indicate a secure boot stage, or perhaps related to some other function.

We can guess now its a MediaTek SOC in this system. Being ARM little endian we might be able to direct lift binaries from a Raspberry Pi, but also possible to cross-compile them using something like the OpenWRT build chain. which might avoid library dependency issues.

Running binwalk a second time and extracting lets us poke around in the Linux filesystem.

```
binwalk redacted.FRM
```

This gives us the following directory structure, after helpfully unpacking the squashfs file (other command line options are needed to get at ("carve") the bootloader and kernel):

```
~/redacted.FRM.extracted/squashfs-root$ ls -ltr
total 24
drwxrwxr-x  7 nuc nuc 4096 May 18  2018 plugins
drwxrwxr-x  7 nuc nuc 4096 May 18  2018 lib
drwxrwxr-x 12 nuc nuc 4096 May 18  2018 usr
drwxrwxr-x  2 nuc nuc 4096 May 18  2018 bin
drwxrwxr-x  2 nuc nuc 4096 May 18  2018 sbin
drwxr-xr-x  9 nuc nuc 4096 May 18  2018 res
```
Squashfs is a read only file system, commonly used as base filesystem in routers and other embedded Linux. It will typically have a writable layer over the top tha lives either in RAM and/or another part of the flash for persistence.

I had a bit of a look around, there are some interesting binaries that might prove useful (BTW, this device is not a Sony):

```
~/redacted.FRM.extracted/squashfs-root$ ls usr/bin/
'['    chrt   dhcpd        dhcpd.leases      du     find   gdbserver   hexdump   lircd_simulator   md5sum    openssl   pkill   test   top   wc        wpa_supplicant               xargs
 awk   curl   dhcpd.conf   dhcpd_sony.conf   expr   gawk   head        killall   lirc_monitor      ntpdate   ping      tee     tftp   tr    wpa_cli   wpa_supplicant.conf.ralink
```

- gdbserver - remote debugging
- openssl gives us a way to download files onto the device if I can find a shell, if all else fails. But curl is there anyway. And TFTP.
- this DVD player cna have accurate time! (ntp)

```
~/redacted.FRM.extracted/squashfs-root$ ls usr/sbin
chroot  inetd  telnetd
```

Ha, how can we get this to run telnetd?

```
~/redacted.FRM.extracted/squashfs-root$ ls bin
basename  cat    cp   date  dfbdump  env    fdisk  gunzip  kill  login  mkdir  mount       mv   nice       ps  rmdir          sed  sleep    sort  tar     umount.cifs
bash      chmod  cut  dd    echo     false  grep   gzip    ln    ls     mknod  mount.cifs  net  nmblookup  rm  secure-script  sh   smbtree  sync  umount  uname
```

Seems we can mount Windows network shares (mount.cifs)

```
~/redacted.FRM.extracted/squashfs-root$ ls sbin
agetty   dhcpc.script    dns.script      getty     ifconfig         insmod  losetup    netinfd    ntfslabel  poweroff      reboot  shutdown    traceroute6  ubiformat  ubinize
arping   dhcpv6.script   flash_erase     halt      ifconfig.script  ip      lsmod      netperf    ping       pppoe.script  rmmod   tcpdump     ubiattach    ubimkvol   ubirmvol
busybox  dibbler-client  flash_eraseall  hostname  init             ipcd    mkfs.ext3  netserver  ping6      rdnss         route   tracepath6  ubidetach    ubinfo     udhcpc
```

They even left tcpdump on it.

I think there is a lot of potential _if_ I can find a way to a shell.

## Open Port Analysis

Given 9222 is chromecast, lets see whats on the other two apparent HTTP servers.

## Security testing (safe for use)

Purpose of this is to check it doesnt phone home, or if it does what to block on my network. They don't need to know when I'm watching a DVD thats my business.

To start with, the plan is to use a spare OpenWRT router as an Ethernet to wifi bridge - that will also let me block and log _everything_ to start with.

I'm intending to describe that in Part 2 of this block, if there is one. I think the family want to watch a movie this weekend, and other activities beckon...