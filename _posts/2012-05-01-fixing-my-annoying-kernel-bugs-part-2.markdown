---
author: admin1
comments: false
date: 2012-05-01 06:28:12+00:00
layout: post
slug: fixing-my-annoying-kernel-bugs-part-2
title: Fixing my annoying kernel bug(s) - Part 2
wordpress_id: 315
categories:
- linux
tags:
- debian
- kernel
---

This blog entry details some of the problem outlined in [these](/2012/patching-and-building-a-custom-linux-kernel-in-debian/) [posts.](2012/fixing-my-annoying-kernel-bugs-part-1/)

This is a lengthy technical post:
**TL;DR**
There is a detailed manual for building a Debian kernel at [http://users.wowway.com/~zlinuxman/Kernel.htm](http://users.wowway.com/~zlinuxman/Kernel.htm).  I was familiar with much of the content already but it was still a very helpful reference; for example, using the 'src' group to avoid root was a useful thing to learn.

The information on patching the Kernel for Phenom was cobbled together from various websites including the Gentoo forums.

My system is currently built from Debian squeeze with a bunch of packages from various other repositories including the Debian [backports](backports.debian.org) and packages I manually backported from Wheezy (testing).

I could have applied the necessary patch to this kernel but I decided at the same time to have another go at getting to the latest 3-series kernel.

For a long time I was stuck on a 2.6.39 kernel as I wasn't able to successfully simply build a later kernel package from the Debian sources that were in testing. I could have tried to build from the kernel.org sources but I have tried where possible to maintain my system using .deb packages as far as possible. In the intervening year however it seems 3.2 has been released in backports, so that saved me a lot of potential problems.

So I upgraded my kernel and applied the patch. Here is Yet Another Tutorial on building a kernel the 'Debian Way'. This will yield a DPKG file that you can install without clobbering any other kernels.



### Prerequisites






  * Install various pre-requisites - this will vary depending on your system.
A fresh system will need many others, I needed these for LZ compression and for 'make xconfig'
[crayon lang="bash"]
sudo aptitude install libqt4-dev=4:4.6.3-4 lzop
[/crayon]

  * 
You need to have the Debian backports in your APT sources.list file. 
[crayon lang="bash"]
deb http://backports.debian.org/debian-backports/ squeeze-backports main contrib non-free
[/crayon]
Having added this, do a `sudo apt-get update`.

  * The ideal method these days is to be able to do most of the work without dropping to root or using sudo.  To achieve this add your account to the 'src' group and setup permissions accordingly.
[crayon lang="bash"]
sudo adduser $LOGNAME src
sudo chmod g+ws /usr/src
sudo chgrp src /usr/src
[/crayon]
At this point you will need to log out and log in (although you could ssh back in as yourself, and I read somewhere recently that this may not be strictly necessary with the right 'magic' incantations any more...)



  * I like to experiment with virtualisation so along the way I downloaded a patch that may be necessary for this from [http://users.wowway.com/~zlinuxman/kernel-package/linuxv3.diff](http://users.wowway.com/~zlinuxman/kernel-package/linuxv3.diff) (This is also an attachment to this post)
Apply like:
[crayon lang="bash"]
cd /usr/src
wget http://users.wowway.com/~zlinuxman/kernel-package/linuxv3.diff
cd /usr/share/kernel-package
sudo patch -b -p1 

  * I also made my own patches to do an optimized build for my AMD64 Phenom:
File phenom_1.patch:
[crayon lang="diff"]
--- Makefile	2012-03-17 21:56:01.786123664 +1030
+++ Makefile	2012-03-17 21:56:03.826122662 +1030
@@ -205,7 +205,7 @@
 
 HOSTCC       = gcc
 HOSTCXX      = g++
-HOSTCFLAGS   = -Wall -Wmissing-prototypes -Wstrict-prototypes -O2 -fomit-frame-pointer
+HOSTCFLAGS   = -march=amdfam10 -Wall -Wmissing-prototypes -Wstrict-prototypes -O2 -fomit-frame-pointer
 HOSTCXXFLAGS = -O2
 
 # Decide whether to build built-in, modular, or both.
[/crayon]
File phenom_2.patch:
[crayon lang="diff"]
--- a/arch/x86/Makefile	2012-03-17 21:52:24.802227892 +1030
+++ b/arch/x86/Makefile	2012-03-17 21:53:06.590208158 +1030
@@ -50,7 +50,7 @@
         KBUILD_CFLAGS += -m64
 
         # FIXME - should be integrated in Makefile.cpu (Makefile_32.cpu)
-        cflags-$(CONFIG_MK8) += $(call cc-option,-march=k8)
+        cflags-$(CONFIG_MK8) += $(call cc-option,-march=amdfam10)
         cflags-$(CONFIG_MPSC) += $(call cc-option,-march=nocona)
 
         cflags-$(CONFIG_MCORE2) += \
[/crayon]
File phenom_3.patch:
[crayon lang="diff"]
--- a/arch/x86/Makefile_32.cpu	2012-03-17 22:03:38.529886664 +1030
+++ b/arch/x86/Makefile_32.cpu	2012-03-17 22:04:08.701869960 +1030
@@ -24,7 +24,7 @@
 # Please note, that patches that add -march=athlon-xp and friends are pointless.
 # They make zero difference whatsosever to performance at this time.
 cflags-$(CONFIG_MK7)		+= -march=athlon
-cflags-$(CONFIG_MK8)		+= $(call cc-option,-march=k8,-march=athlon)
+cflags-$(CONFIG_MK8)		+= $(call cc-option,-march=amdfam10)
 cflags-$(CONFIG_MCRUSOE)	+= -march=i686 $(align)-functions=0 $(align)-jumps=0 $(align)-loops=0
 cflags-$(CONFIG_MEFFICEON)	+= -march=i686 $(call tune,pentium3) $(align)-functions=0 $(align)-jumps=0 $(align)-loops=0
 cflags-$(CONFIG_MWINCHIPC6)	+= $(call cc-option,-march=winchip-c6,-march=i586)
[/crayon]



  * And of course, the patch to fix my Firewire subsystem crash as described in the previous post:
[crayon lang="diff"]
--- a/block/bsg.c.orig	2012-03-17 22:06:42.053784045 +1030
+++ b/block/bsg.c	2012-03-17 22:07:01.037773291 +1030
@@ -985,7 +985,8 @@
 
 	mutex_lock(&bsg;_mutex);
 	idr_remove(&bsg;_minor_idr, bcd->minor);
-	sysfs_remove_link(&q-;>kobj, "bsg");
+	if (q->kobj.sd)
+	        sysfs_remove_link(&q-;>kobj, "bsg");
 	device_unregister(bcd->class_dev);
 	bcd->class_dev = NULL;
 	kref_put(&bcd-;>ref, bsg_kref_release_function);
[/crayon]




### Procedure






  1. Install the Debian source package and unpack the tree:
[crayon lang="bash"]
sudo apt-get install linux-source-3.2 linux-image-3.2
cd /usr/src
tar xjf linux-source-3.2.tar.bz2
[/crayon]


Note - it turns out that this is in fact a 3.2.9 kernel.  For some reason the Debian version is 3.2.4-1~bpo60+1 ; go figure...

  2. Apply patches: assumes the patch files are in /usr/src :
[crayon lang="bash"]
cd /usr/src/kernel-source-3.2
patch < ../amm_phenom_1.patch
patch -p1 < ../amm_phenom_2.patch
patch -p1 < ../amm_phenom_3.patch
patch -p1 < ../amm_bsg.patch
[/crayon]



  3. Configure the kernel build:
I started by copying the default config from the binary backports kernel, and tweaking it for my own purposes (not shown here)
[crayon lang="bash"]
cd /usr/src/linux-source-3.2
cp /boot/config-3.2.0-0.bpo.1-amd64 .config
make silentoldconfig
make xconfig
make-kpkg --rootcmd fakeroot modules_clean
make-kpkg clean
[/crayon]



  4. Finally, build the Debian packages.
[crayon lang="bash"]
CONCURRENCY_LEVEL=4 time fakeroot make-kpkg --initrd \
        --append-to-version=-xxx-preempt-amd64 --revision 1~yyy.00.00 \
        kernel_image kernel_headers kernel_debug
[/crayon]
Here, CONCURRENCY_LEVEL=4 sets the number of concurrent make processes used, for taking advantage of a multi-core system.
From the above settings, the actual package will become 'linux-image-3.2.9-xxx-preempt-amd64' with a Debian version of '1~yyy.00.00' and `cat /proc/version` output of '3.2.9-xxx-preempt-amd64'
Using this mechanism means you can have concurrent 'flavours' of a kernel installed but still being upgradable within that flavour.



  5. Installation:
[crayon lang="bash"]
cd /usr/src
sudo dpkg -i linux-image-3.2.9-xxx-preempt-amd64_1~yyy.00.00_amd64.deb linux-headers3.2.9-xxx-preempt-amd64_1~yyy.00.00_amd64.deb
[/crayon]
This should also trigger any DKMS modules to rebuild if present.
My NVidia 280.13 driver rebuilds fine with this version.





### Testing


Of course the proof is in the pudding.

After rebooting, I repeated the sequence necessary to trigger the fault: and it did not recur. Woot!

Attachments:
[linuxv3.diff](/wp-content/uploads/2012/05/linuxv3.diff)
