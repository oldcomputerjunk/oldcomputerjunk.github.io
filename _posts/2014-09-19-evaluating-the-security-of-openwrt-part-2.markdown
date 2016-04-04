---
author: admin1
comments: false
date: 2014-09-19 14:37:11+00:00
layout: post
slug: evaluating-the-security-of-openwrt-part-2
title: Evaluating the security of OpenWRT (part 2)
wordpress_id: 557
categories:
- infosec
tags:
- embedded
- hacking
- openwrt
- security
---

In my last post I covered how I setup an OpenWRT build, to examine  a small subset of indicators of security of the firmware.

In this follow-up post we will examine in detail the analysis results of one of the indicators: specifically, the RELRO flag.


## A first look - what are the defaults?


The analysis here is specific to the Barrier Breaker release of OpenWRT, but it should be noted that during experiments with the OpenWRT development trunk the results are much the same.

Before diving into RELRO, lets take a look at the overall default situation.

Here is the checksec report for the Carambola2 device (MIPS processor) build.
It is a sea of red...

[![Default checksec report for OpenWRT x86 build](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/09/owrt_cs_report_0_c2.png)](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/09/owrt_cs_report_0_c2.png)The 'run as root' errors can be ignored: those programs are actually absolute symbolic links which do not resolve in the host system. Relative symbolic links resolve correctly but are filtered out of the analysis.

The x86 build paints a similar picture:

[![owrt_cs_report_0_x86](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/09/owrt_cs_report_0_x86.png)](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/09/owrt_cs_report_0_x86.png)(Notably for x86, the NX flag is correctly set, but that is a topic for another time.)

Note, the rest of this post describes how to modify OpenWRT to enable RELRO.  There may be perfectly valid reasons to not enable the flag (for example, using RELRO may have a performance impact, and for a given system the adverse security risk may be judged low), so I have ensured that the suggested mitigation if applied remains a choice in the configuration menu of the system.  For the moment my patch also retains backward compatibility by defaulting to off.


## Inside the OpenWRT build system


After a brief look at the build logs, the reason is obvious: the typical gcc linker command is missing the flags needed to enable RELRO:  `-Wl,-z,relro -Wl,-z,now` (or the direct linker equivalents, `-z relro -z now`)

What could be done to address this?

OpenWRT provides a hook for appending to the global compiler `CFLAGS` but there is no similar hook for the linker stage. We could add those flags to the global `CFLAGS` and they can in fact flow through to the linker for many programs, but that would also be redundant as the flags are irrelevant to the compiler.  In the end I decided I would modify the OpenWRT source to add a new global CONFIG option, which adds `-Wl,-z,relro -Wl,-z,now` to the global `LDFLAGS` instead.

The following patch achieves that (note, I have left out some of the help for brevity):
[crayon lang="text"]diff --git a/rules.mk b/rules.mk
index c9efb9e..e9c58d8 100644
--- a/rules.mk
+++ b/rules.mk
@@ -177,6 +177,10 @@ else
   endif
 endif
 
+ifeq ($(CONFIG_SECURITY_USE_RELRO_EVERYWHERE),y)
+  TARGET_LDFLAGS+= -Wl,-z,relro -Wl,-z,now
+endif
+
 export PATH:=$(TARGET_PATH)
 export STAGING_DIR
 export SH_FUNC:=. $(INCLUDE_DIR)/shell.sh;
diff --git a/toolchain/Config.in b/toolchain/Config.in
index 7257f1d..964200d 100644
--- a/toolchain/Config.in
+++ b/toolchain/Config.in
@@ -38,6 +38,19 @@ menuconfig TARGET_OPTIONS
 
                  Most people will answer N.
 
+       config SECURITY_USE_RELRO_EVERYWHERE
+               bool "Enable RELRO and NOW for binaries and libraries" if TARGET_OPTIONS
+               default n
+               help
+                 Apply -z relro -z now flag to the linker stage for all ELF binaries and libraries.
 
 menuconfig EXTERNAL_TOOLCHAIN
        bool
[/crayon]
Having attched OpenWRT, and enabled the new flag, lets rebuild everything again and run another checksec scan.

[![owrt_cs_report_1_x86](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/09/owrt_cs_report_1_x86.png)](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/09/owrt_cs_report_1_x86.png)What a difference!

The results shown above are for x86, the picture is similar for the Carambola2 MIPS image.

The new results indicate that the RELRO flag is present on some binaries but not all of them. From this we can predict that some packages do not fully honour the global OpenWRT build system linker flags. I soon confirmed this**; the implication is that the new flag CONFIG_SECURITY_FORCE_RELRO  is useful, however, a caveat in the Kconfig help is required. In particular, a statement to the effect that the efficacy depends on proper coding of OpenWRT packages (with ideally all packages maintained by the project being fixed to honour the flag.)

** For example: the package that builds libnl-tiny.so does not pass LDFLAGS through to the linker; this and some other base system packages needed patching to get complete coverage.  it is likely that there are other packages that I did not have selected that may also need tweaking.

Another notable package is busybox.  Busybox it turns out uses `ld` directly for linking, instead of indirectly via gcc, and thus requires the flags in the pure form `-z relro -z now`.  (The busybox OpenWRT package Makefile also happens to treat the global TARGET_LDFLAGS differently from the TARGET_CFLAGS although I am unsure if this is a bug; but that turned out to be a red-herring.)  Oddly, this solution worked for MIPS when I tried it previously, but is presently not successful for the x86 build, so further investigation is needed here; possibly I incorrectly noted the fix in previous experiments.



## Fun and Games with uClibc and busybox


The other recalcitrant is the uClibc library. I spent quite a bit time trying to work out why this was not working, especially having  confirmed with verbose logging that the flags are being applied as expected. Along the way I learned that uClibc already has its own apply RELRO config item, which was already enabled.  Even more oddly, RELRO is present on some uClibc libraries and not others, that as far as I could tell were being linked with identical linker flag sets.

After some digging I discovered hints of bugs related to RELRO in various versions of binutils, so I further patched OpenWRT to use the very latest binutils release.   However that made no difference.  At this point I took a big diversion and spent some time building the latest uClibc externally, where I discovered that it built fine using the native toolchain of Debian Wheezy (including a much older binutils!)  After some discussion on the uClibc mailing list I have come to the conclusion that there may be a combination of problems, including the fact that uClibc in OpenWRT is a couple of years old (and additionally has a set of OpenWRT specific patches.)   I could go further and patch OpenWRT to use the trunk uClibc but then I would have to work through refreshing the set of patches which I really don't have time or inclination to do, so for the moment I have deferred working on resolving this conundrum.  Eventually someone at OpenWRT may realise that uClibc has undergone a flurry of development in recent times and may bump to the more recent version.


## Comments


Along the way, I discovered that Debian actually runs security scans across all packages in the distribution - take a look at [https://lintian.debian.org/tags/hardening-no-relro.html](https://lintian.debian.org/tags/hardening-no-relro.html).

It is worth noting that whenever changing any build-related flag it is worth cleaning and rebuilding the toolchain as well as the target packages and kernel; I found without doing this, flag changes such as the RELRO flag don't fully take effect as expected.

For maximum verboseness, run with `make V=csw` although I had to dig through the code to find this out.

I was going to repeat all the testing against a third target, another  MIPS-based SOC the  RALINK 3530 but at this point I don't really have the time or inclination, I am sure the results will be quite similar.  It would probably be useful to try with an ARM-based target as well.

I should also try repeating this experiment with MUSL, which is an alternative C library that OpenWRT can be built with.


## Conclusion


Out of the box, OpenWRT has very limited coverage of the RELRO security mitigation in a standard firmware build.  By applying the suggested patches it is possible to bring OpenWRT up to a level of coverage, for RELRO, to that approaching a hardened Gentoo or Ubuntu distribution, with only a small subset of binaries missing the flag.


### References


My Github account includes the repository [openwrt-barrier-breaker-hardening](https://github.com/pastcompute/openwrt-barrier-breaker-hardening). The following branch include the completed series of patches  mentioned above:  [owrt_analysis_relro_everywhere](https://github.com/pastcompute/openwrt-barrier-breaker-hardening/tree/owrt_analysis_relro_everywhere)  I hope it will remain possible to apply these changes against the official release for a while yet.

The patch that enables the latest binutils is not in that branch, but in this [commit](https://github.com/pastcompute/openwrt-barrier-breaker-hardening/commit/f71f1e85266d6b6afd076b764693042aa1261e9c).
