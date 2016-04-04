---
author: admin1
comments: false
date: 2014-10-03 15:05:18+00:00
layout: post
slug: evaluating-the-security-of-openwrt-part-3-adventures-in-noexecstackland
title: Evaluating the security of OpenWRT (part 3) adventures in NOEXECSTACK'land
wordpress_id: 580
categories:
- infosec
- linux
tags:
- embedded
- openwrt
- security
---

To recap our experiments to date, out of the box OpenWRT, with further digging, may _appear to give the impression_ to have sporadic coverage of various Linux hardening measures without doing a bit of extra work. This in fact can _**be a false impression**_ - see [update](http://blog.oldcomputerjunk.net/2014/update-evaluating-the-security-of-openwrt-part-3-adventures-in-noexecstackland/) - but for the uninitiated it could take a bit of digging to check!
One metric of interest not closely examined to date is the NOEXECSTACK attribute on executable binaries and libraries. When coupled with Kernel support, if enabled this disallows execution of code in the stack memory area of a program, thus preventing an entire class of vulnerabilities from working.  I mentioned NOEXECSTACK in passing previously; from the checksec report we saw that the x86 build has 100% coverage of NOEXECSTACK, whereas the MIPS build was almost completely lacking.

For a quick introduction to NOEXECSTACK, see [http://wiki.gentoo.org/wiki/Hardened/GNU_stack_quickstart](http://wiki.gentoo.org/wiki/Hardened/GNU_stack_quickstart).


## Down the Toolchain Rabbit Hole


As far as a challenging detective exercise, this one was a bit of a doosy, for me at least.  Essentially  I had to turn the OpenWRT build system inside out to understand how it worked, and then the same with uClibc, so that I could learn where to begin to start.  After rather a few false starts, the culprit turned out to be somewhere completely different.

First, OpenWRT by default uses uClibc as the C library, which is the bedrock upon which the rest of the user space is built.  The C library is not just a standard package however. OpenWRT, like the majority of typical embedded Linux systems employs a  "toolchain" or "buildroot" architecture.  Simply put, the combines packages together the C/C++ compiler, the assembler, linker, the C library and various other core components in a way that the remainder of the firmware can be built without having knowledge of how this layer is put together.

This is a Good Thing as it turns out, especially when cross-compiling, i.e. when building the OpenWRT firmware for a CPU or platform (the TARGET) that is different from that where the firmware build happens (the HOST.)  Especially where everything is bootstrapped from source, as OpenWRT is.

The toolchain is combined from the following components:



	
  * binutils -- provides the linker (`ld`) and the assembler and code for manipulating ELF files (Linux binaries) and object libraries

	
  * gcc -- provides the C/C++ compiler and, often overlooked, libgcc, a library various "intrinsic" functions such as optimisations for certain C library functions, amongst others

	
  * A C library -- in this case, uClibc

	
  * gdb -- a debugger


Now all these elements need to be built in concert, and installed to the correct locations, and to complicate matters, the toolchain actually has multiple categories of output:

	
  * Programs that run on the HOST that produce programs and libraries that run on the TARGET (such as the cross compiler)

	
  * Programs that run on the on the TARGET (e.g. `ldd`, used for scanning dependencies)

	
  * Programs and libraries that run on the HOST to perform various tasks related to the above

	
  * Header files that are needed to build other programs that run on the HOST to perform various tasks related to the above

	
  * Header files that are needed to build programs and libraries that run on the TARGET

	
  * Even, programs that run on the on the TARGET  to produce programs and libraries that run on the TARGET (a target-hosted C compiler!)


Confused yet?

All this magic is managed by the OpenWRT build system in the following way:

	
  * The toolchain programs are unpacked and built individually under the build_dir/toolchain directory

	
  * The results of the toolchain build designed to run on the host under the staging_dir/toolchain

	
  * The partial toolchain under staging_dir is used to build the remaining items under build_dir which are finally installed to staging_dir/target/blah-rootfs

	
  * (this is an approximation, maybe build OpenWRT for yourself to find out all the accurate naming conventions )

	
  * The kernel headers are an intrinsic part of this because of the C library, so along the way a pass over the target Linux kernel source is required as well.


OpenWRT is flexible enough to allow the C compiler to be changed (e.g. between stock gcc 4.6 and LInaro gcc 4.8) , and the binutils version, and even switch the C library between different project implementations ( uClibc vs eglibc vs MUSL.)

OpenWRT fetches the sources for all these things, then applies a number of local patches, before building.

We will need to refer to this later.


## Confirming the Problem and Fishing for Red Herrings.


The first thing to note is that x86 has no problem, but MIPS does, and I want to run OpenWRT on various embedded devices with MIPS SOC.  Without that I may never have bothered digging deeper!

Of course I dived in initially and took the naive brute force approach.  I patched OpenWRT to apply the override flag to the linker: `-Wl,-z,noexecstack`. This was a bit unthinking, after all x86 did not need this.

Doing this gave partial success. In fact most programs gained NOEXECSTACK, except for a chunk of the uClibc components, busybox, and tellingly as it turned out, libgcc_s.so. That is, core components used by nearly everything. Of course.

(_**Spoiler: modern Linux toolchain implementations actually enable NOEXECSTACK by DEFAULT for C code!**_ Which was an important fact I forgot at this point! Silly me.)

At this point,  I managed to overlook libgcc_s.so and decided to focus on uClibc. This decision would greatly expand my knowledge of OpenWRT and uClibc and embedded built systems, and do nothing to solve the problem the proper way!

OpenWRT builds uClibc as a host package, which basically means it abuses Makefiles to generate a uClibc configuration file partly derived from the OpenWRT config file settings, and eventually call the uClibc top level makefile to build uClibc. This can only be done after building binutils and two of three stages of gcc.

At this point I still did not fully understand how NOEXECSTACK really should be employed, which is probably an artefact of working on this stuff late at night and not reading everything as carefully as I might have.  So I did the obvious and incorrect thing and worked out how to patch uClibc further to push the force `-Wl,-z,noexecstack` through it. What I had to do to do that could almost take another blog article, so I'll skip it for brevity.  Anyway, this did not solve the problem.

Finally I turned on all the debug and examined the build:

    
    make V=csw toolchain/uClibc/compile


(Aside: the documentation for OpenWRT mentions using `V=s` to turn on some verboseness, but to get the actual compiler and linker commands of the toolchain build you need the extra flags. I should probably try and update the OpenWRT wiki but I have invested so much time in this that I might have to leave that as an exercise for the reader)

All the libraries were being linked using the `-Wl,-z,noexecstack` flag, yet some still failed checksec. Argh!

I should also note that repeating this process over and over gets tedious, taking about 20 minutes to built the toolchain plus minimal target firmware on my quad core AMD Phenom. Dont delete the build_dir/host and staging_dir/host  directories or it doubles!

So something else was going on.


## Upgrades and trampolines, or not.


I sought help from the uClibc developers mailing list, _**who suggested I first try using up to date software. This was a fair point,**_ as OpenWRT is using a 2 year old release of uClibc and 1 year old release of binutils, etc.

This of course entailed having to learn how to patch OpenWRT to give me that choice.

So another week later, around work and family and life, I found some time to do this, and discovered that the problem persisted.

At this point I revisited the Gentoo hardening guide.  After some detective work I discovered that several MIPS assembler files inside of uClibc did not actually have the recommended code fragments. Aha! I thought. Incorrectly again, as I should have realised; uClibc has already thought of this and when NOEXECSTACK is configured, as it is for OpenWRT, uClibc passes a different flag to the assembler that has the effect of fixing NOEXECSTACK for assembler files. And of course after I patched about 17 .S files and waited another half hour, the checksec scan was still unsuccessful. Red herring again!

I started to get desperate when I read about some compiler systems that use something called a 'trampolline'. So I went back to the mailing uClibc  list.

_**At this point I would like to thank the uClibc developers for being so patient with me**_, as the solution was now near at hand.


## Cutting edge patches and a wrinkle in time.


One of the uClibc developers pointed me to a patch posted on the gcc mailing list. As fate would happen, dated [10 September 2014](https://gcc.gnu.org/ml/gcc-patches/2014-09/msg02430.html), which was _after_ I started on these investigations.  This actually went full circle back to libgcc_s.so which was the small library I passed over to focus on uClibc.  This target library itself has some assembly files, which were neither built with the noexecstack option nor including the Gentoo-suggested assembly pragma. This patch also applies on gcc, not on uClibc, and of course was outside of binutils which was the other component I had to upgrade.  The fact that libgcc_s.so was not clean should maybe have pointed me to look at gcc, and it did cross my mind. But we all have to learn somehow.  Without all the above I would be the poorer for my knowledge of the internals of all these systems.

So I applied this patch, and finally, bliss, a sea of green NX enabled flags. Without in the end any modifications actually required to the older version of uClibc used inside OpenWRT.

This even fixed busybox NX status without modification.  So empirically this confirms what I read previously and also overlooked to my detriment, that being NOEXECSTACK is aggregated from all linked libraries: if one is missing it pollutes the lot.


## [![Fixed NOEXECSTACK on uClibc](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/10/Selection_031.png)](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/10/Selection_031.png)




## Postscript


Now I have to package the patch so that it can be submitted to OpenWRT, initially against Barrier Breaker given that has just been released!

Then I will probably need to repeat it against all the supported gcc versions and submit separate patches for those. That will get a bit tedious, thankfully I can start a test build then go away...
