---
author: admin1
comments: false
date: 2014-09-18 13:18:30+00:00
layout: post
slug: evaluating-the-security-of-openwrt-part-1
title: Evaluating the security of OpenWRT (part 1)
wordpress_id: 551
categories:
- infosec
tags:
- embedded
- hacking
- openwrt
- security
---

I have recently spent  some time pondering the state of embedded system security.
I have been a long time user of OpenWRT, partly based on the premise that it should be "more secure" out of the box (provided attention is paid to [routine](http://wiki.openwrt.org/doc/howto/secure.access) hardening); but as the Infosec world keeps  on evolving, I thought I would take a closer look at the state of play.


## OpenWRT - an embedded Linux distribution.


OpenWRT is a Linux distribution targeted as a replacement firmware for consumer home routers, having been named after the [venerable Linksys WRT54g](http://wiki.openwrt.org/about/history), but it is gaining a growing user base in the so-called Internet of Things space, as evidenced by devices such as the [Carambola2](http://8devices.com/carambola-2), [VoCore](https://www.indiegogo.com/projects/vocore-a-coin-sized-linux-computer-with-wifi) and [WRTnode](http://wrtnode.com/).

Now there are many, many areas that could be the focus of security related attention.  One useful reference is the [Arch Linux security guide](https://wiki.archlinux.org/index.php/Security) , which covers a broad array of mitigations, but for some reason I felt like diving into the deeper insides of the operating system. I decided it would be worthwhile to compare the security performance of the core of OpenWRT, specifically the Linux kernel plus the userspace toolchain, against other "mainstream" distributions.

The theory is that ensuring a baseline level of protection in the toolchain and the kernel will cut of a large number of potential root escalation vulnerabilities before they can get started.  To get started, take a look at the [Gentoo Linux toolchain hardening guide](http://wiki.gentoo.org/wiki/Hardened/Toolchain).  This lists mitigations such as: Stack Smashing Protection (SSP), used to make it much harder for malware to take advantage of stack overrun situations; position independent code (PIC or PIE) that randomises the address of a program in memory when it is loaded; and so-on to other features such as Address Space Layout Randomisation (ASLR) in the kernel, all of these designed to halt entire classes of exploits in their tracks.

I wont repeat all the pros and cons of the various methods here.  It should be noted that these methods are not foolproof; new techniques such as Return Oriented Programming (ROP) can be used to circumvent these mitigations.  But defense-in-depth would appear to be an increasingly important strategy.  If a device can be made secure using these techniques it is probably ahead of 95% of the embedded market and more likely to avoid the larger number of automated attacks; perhaps this a bit like the old joke:


**The first zebra said to the second zebra: "Why are you bothering to run? The cheetah is faster than us!" **




**To which the second zebra replied, "I only have to run faster than you!"**





## Analysis of just a few aspects.


There is an excellent tool "checksec.sh", that can scan binaries in a Linux system and report their status against a variety of  security criteria.
The original author has retained his [website](http://www.trapkit.de/tools/checksec.html) but the script was last updated in November 2011. I found a more recently maintained version on [GitHub](https://github.com/slimm609/checksec.sh) featuring various enhancements (the maintainer even accepted a patch from me addressing a cosmetic bugfix.)

Initially I started using the tool to report on just the status of the following elements in binaries and libraries: [RELRO](http://tk-blog.blogspot.com.au/2009/02/relro-not-so-well-known-memory.html), [non-executable stack](http://blog.siphos.be/2011/07/high-level-explanation-on-some-binary-executable-security/), SSP and PIC/PIE.

The basic procedure for using the tool against an OpenWRT build is straightforward enough:



	
  1. Configure the OpenWRT build and ensure the tar.gz target is enabled ( `CONFIG_TARGET_ROOTFS_TARGZ=y` )

	
  2. To expedite the testing I build OpenWRT with a pretty minimal set of packages

	
  3. Build the image as a tar.gz

	
  4. Unpack the tar.gz image to a temporary directory

	
  5. Run the tool


When repeating a test I made sure I cleaned out the toolchain and the target binaries, because I ahve been bitten by spuriuos results caused by changes to the configuration not propagating through all components.  This includes manually removing the staging_dir which may be missed by `make clean`. This made each build take around 20+ minutes on my quad core Phenom machine.

I used the following base configuration, only changing the target:
```Makefile
CONFIG_DEVEL=y
CONFIG_DOWNLOAD_FOLDER="/path/to/downloads"
CONFIG_BUILD_LOG=y
CONFIG_TARGET_ROOTFS_TARGZ=y
```

I repeated the test for three platform configurations initially (note, these are mutually exclusive choices) - the Carambola2 and rt305x are MIPS platforms.
```Makefile
CONFIG_TARGET_ramips_rt305x_MPRA1=y
CONFIG_TARGET_x86_kvm_guest=y
CONFIG_TARGET_ar71xx_generic_CARAMBOLA2=y
```

There are several directories most likely to have binaries of interest, so of course I scanned them using a script, essentially consisting of:

```bash
for p in lib usr/lib sbin usr/sbin bin usr/bin ; do checksec.sh --dir $p ; done```

The results were interesting.

(to be continued)


### Further reading


[1] [http://wiki.openwrt.org/doc/howto/secure.access](http://wiki.openwrt.org/doc/howto/secure.access)
[2] [https://wiki.archlinux.org/index.php/Security](https://wiki.archlinux.org/index.php/Security)
[3] [http://wiki.gentoo.org/wiki/Hardened/Toolchain](http://wiki.gentoo.org/wiki/Hardened/Toolchain)
[4] [http://blog.siphos.be/2011/07/high-level-explanation-on-some-binary-executable-security/](http://blog.siphos.be/2011/07/high-level-explanation-on-some-binary-executable-security/)
[5] [http://www.trapkit.de/tools/checksec.html](http://www.trapkit.de/tools/checksec.html)
