---
author: admin1
comments: false
date: 2014-11-30 12:03:33+00:00
layout: post
slug: running-freebsd-on-the-carambola2
title: Running FreeBSD on the carambola2
wordpress_id: 590
categories:
- tech
tags:
- carambola2
- embedded
- freebsd
---

The carambola2 is a small module built around the Atheros AR9330 SOC. Manufactured by [8devices](http://www.8devices.com/carambola-2), it has 64MB RAM, 16MB flash, two Ethernet ports and a host of GPIO pins, some of which can be configured as i2c, SPI or i2s. The carambola2 is shipped with [OpenWRT](http://openwrt.org/) a Linux distribution targeted at small devices and as a replacement firmware for consumer routers.

I have previously presented on using the carambola2 at the [Sysadmin Miniconference](http://sysadmin.miniconf.org/presentations14.html#AndrewMcDonnell) at LCA2014 (slides [here](http://andrewmcdonnell.net/slides/lca2014_sysadmin_talk.pdf), video [here](http://mirror.linux.org.au/linux.conf.au/2014/Monday/167-Custom_equipment_monitoring_with_OpenWRT_and_Carambola_-_Andrew_McDonnell.mp4))

To try something different, I thought I'd take [FreeBSD](https://www.freebsd.org/) for a run on this board. This became an extensive learning exercise as I knew absolutely nothing about any of the *BSD distributions other than their unix heritage and that they use BSD type licenses instead of GPL for the kernel and most of the userland.


## Preparation


As with any of these things, there are a bunch of perceived or actual pros/cons between OpenWRT and FreeBSD.
Some of these I only discovered during this process.

My requirements included:



	
  * Being able to do a complete firmware build from source, which is possible for both OpenWRT and FreeBSD

	
  * Easy access to LED and GPIOs

	
  * Run the image from a RAM filesystem


Some pros/cons of either include:

	
  * The build system used is actually the standard build system for FreeBSD. You could probably build OpenWRT under OpenWRT but you usually dont.

	
  * The build system when used for cross compiling is functional but not as elegant as OpenWRT

	
  * OpenWRT builds actually take significantly longer from scratch for some reason

	
  * FreeBSD may be regarded as more secure under some circumstances, for some definition of security. But see below...

	
  * FreeBSD ships with two firewalls: pf, and ipfw. This adds quite a learning curve when doing a bottom up build like this.

	
  * Many common packages (the BSD "ports" system) do not cross-build correctly for mips under FreeBSD

	
  * FreeBSD 10.x ships with llvm as the default compiler but falls back to gcc for cross-building mips. But the gcc supplied with FreeBSD is only 4.2      O_o      Apparently this is for licensing reasons. This can be [worked around](https://www.freebsd.org/doc/en_US.ISO8859-1/articles/custom-gcc/article.html)  but I haven't had time to try it yet. Ramifications of this likely include weaker security.


The carambola2 and the mips platform in general is actually reasonably [well supported](https://wiki.freebsd.org/FreeBSD/mips/Carambola2) by FreeBSD, although it is treated as a 'beta'. As to be expected, to build a firmware for FreeBSD requires a host FreeBSD system (at least this would be the path of least resistance!)

I built a virtual machine using kvm and was able to install FreeBSD 10.0 with minimum of hassle. I was pleasantly surprised at how easy it was to get up and running to OpenBox. FreeBSD has 'pkg' as a binary package manager and it worked similarly enough to 'apt-get', or 'yum' that I had a build machine up in about half an hour.
I did need to install bash and vim and gedit, some things are just too hard to give up!


## Build Process


There appeared to be more than one way to cross-build, including the use of qemu as a build host inside FreeBSD, but rather than chasing turtles on this occasion I went with a tool called '[freebsd-wifi-build](https://github.com/freebsd/freebsd-wifi-build)'. This was actually quite straightforward and produced me a working firmware out of the box, with some caveats. The firmware includes only binaries from the FreeBSD base userland, and only a limited subset at that. Initially it also wanted to build as the root user, which was both an annoyance and a shock to discover, although I soon resolved that problem; I hope to soon have patches accepted into the project to change the default to build as user!

In general, constructing a firmware using FreeBSD is more manual than OpenWRT, as it lacks the all-encompassing configuration of packages and the packaging infrastructure provided by OpenWRT opkg. It is more  similar to the Linux [buildroot](http://buildroot.uclibc.org) or even Gentoo.

The end result is a build script that automates the process I used to customise things, this is published at [https://github.com/pastcompute/carambola2-freebsd-userbuild](https://github.com/pastcompute/carambola2-freebsd-userbuild) , for use as you see fit.

To flash the firmware, I used scp to copy the image to my host machine then using minicom to connect to the board, flash via tftp.
freebsd-wifi-build produces separate kernel and filesystem images, I was able to combine them into one file to simplify flashing.


## Easy Wins





	
  * Network worked, with caveats

	
  * I was able to toggle the LED using '/dev/led', although overall Linux has much better access to GPIO / LED hardware




## Tweaking required along the way





	
  * FreeBSD swaps the ethernet ports relative to OpenWRT, and also by default configured them in switched mode instead of independently routed. I resolved this by rebuilding the firmware with the latest FreeBSD kernel from -CURRENT, which made the ethernet PHY configuration configurable.

	
  * As part of resolving that, I by chance discovered I could built the FreeBSD-release-10.1.0 userland and the bleeding edge FreeBSD-CURRENT kernel and have them cooperate together!

	
  * Only some of the FreeBSD ports easily build with the default cross compiler configuration. This limits the software that can be installed (at least, if built using the ports infrastructure)

	
  * Defaulting to gcc-4.2 means various important security measures, such as -fstack-protector, are disabled

	
  * I also had to tweak the default FreeBSD kernel configuration provided for the carambola2, to turn on the FAT filesystem (for USB transfer) and to enable additional GPIO

	
  * FreeBSD ignores uboot environment and arguments on the ar71xx platform, I managed to patch the kernel to support that




## Summary


I'll keep using OpenWRT on most of my devices for the forseeable future. But I will have a couple of FreeBSD gadgets thrown into the mix, just so I can keep learning new things, and also because ironically FreeBSD supports another router I have, the dir-632 ( I [blogged ](http://blog.oldcomputerjunk.net/2013/achievement-unlocked-openwrt-device-porting-and-a-quick-and-dirty-level-conversion/)about this device previously) which is not officially supported in the mainline OpenWRT and probably wont be anytime soon, but does work in a FreeBSD fork, [zrouter](http://zrouter.org).
It will also be interesting to compare the performance of pf against iptables.

Potential future exploration ideas: running Debian kFreeBSD on the carambola2.
