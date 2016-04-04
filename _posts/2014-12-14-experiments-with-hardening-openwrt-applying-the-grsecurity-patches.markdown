---
author: admin1
comments: false
date: 2014-12-14 13:02:09+00:00
layout: post
slug: experiments-with-hardening-openwrt-applying-the-grsecurity-patches
title: 'Experiments with hardening OpenWRT: applying the grsecurity patches'
wordpress_id: 594
categories:
- infosec
tags:
- embedded
- kernel
- linux
- openwrt
- security
---

A well known set of security enhancements to the Linux kernel is the [grsecurity](https://grsecurity.net/) patch.  The grsecurity patch is a (large) patch that applies cleanly against selected supported stock Linux kernel versions. It brings with it PAX, which protects against various well known memory exploits, plus  a number of other hardening features including logging time and mount changes.<del> In particular it enables features such as Non-executable stack (NX) on platforms that do not provide NX in hardware, such as MIPS devices and older x86.
</del>_**UPDATE**_ _Unfortunately, NX protection for MIPS 32-bit devices is not in fact supported in software. This would be very useful. Whilst I was teaching myself I managed to mix things up, so be aware when reading the rest of this blog entry. Otherwise, the usefulness of grsecurity and the mechanism for patching into OpenWRT is still valid._

_Note also, a more detailed procedure you can use to rebase the patches is at https://github.com/pastcompute/openwrt-cc-ar71xx-hardened/wiki ._


## OpenWRT hardening


OpenWRT is a widely adopted embedded / router Linux distribution. It would benefit greatly from including grsecurity, in particular given most MIPS platforms do not support NX protection in hardware. However for a long time the differences between the OpenWRT kernel and the kernel revisions that grsecurity is supported on have been significant and would likely have taken an extreme effort to get working, let alone get working securely.

This is a shame, because there is [malware](http://securelist.com/analysis/36396/heads-of-the-hydra-malware-for-network-devices/) targeted at [consumer](http://www.computerworld.com/article/2487791/malware-vulnerabilities/-the-moon--worm-infects-linksys-routers.html) embedded [routers](http://threatpost.com/blackenergy-malware-plug-ins-leave-trail-of-destruction/109126), and it must only be a matter of time before OpenWRT is targeted.  OpenWRT is widely regarded as relatively secure compared to many consumer devices, at least if configured properly,  but eventually some bug will allow a remote binary to be dropped. It would be helpful if the system can be hardened and stay one step ahead of things.

The OpenWRT development trunk (destined to become the next release, 'Chaos Calmer' in due course) has recently migrated most devices to the 3.14 kernel tree.  Serendipidously this aligns with the long term supported grsecurity revision 3.14.  When I noticed this I figured I'd take a look at whether it was feasible to deploy grsecurity with OpenWRT.


## Applying grsecurity - patch


In late November I pulled the latest OpenWRT sources and the kernel version was 3.14.25, which I noticed matched the current grsecurity stable branch 3.14.25

The grsecurity patch applies cleanly against a stock kernel, and OpenWRT starts with a stock kernel and then applies a series of patches designed to extend hardware support to many obscure embedded things not present in the mainline kernel, along with patches that reduce the memory footprint. Some of the general patches are pushed upstream but may not yet have been accepted, and some could be backports from later kernels.  Examples of generic patches  include a simplified crash report.

Anyway, I had two choices, and tried them both: apply grsecurity, then the OpenWRT patches; or start with the OpenWRT patched kernel.  In both cases there were a number of rejects, but there seemed to be less when I applied grsecurity last. I also decided this would be easier for me to support for myself going forward, a decision later validated successfully.

OpenWRT kernel patches are stored in two locations; generic patches applying against any platform, then platform specific patches.  My work is tested against the Carambola2, an embedded MIPS board supported by the 'ar71xx' platform in OpenWRT, so for my case, there were ar71xx patches.

To make life easy I wrote a script that would take a directory of OpenWRT kernel patches, apply to a git kernel repository and auto-commit. This allowed me to use gitg and git difftool to examine things efficiently.  It also worked well with using an external kernel tree to OpenWRT so I didnt have to worry yet about integrating patches into OpenWRT. This script is on [github](https://github.com/pastcompute/wrt-buildroot-manager/raw/master/apply-owrt-patches.sh), it should be easily adaptable for other experiments.

(Note: to use an external tree, managed by git, use config options like the following:
[crayon lang="plain"]
CONFIG_KERNEL_GIT_CLONE_URI="path/to/linux-stable"
CONFIG_KERNEL_GIT_LOCAL_REPOSITORY="path/to/linux-stable"
CONFIG_KERNEL_GIT_BRANCH="owrt_grsec_v3.14.25"
[/crayon]

There were four primary rejects that required fixing.  This involved inspecting each case and working out what OpenWRT had changed in the way. Generally, this was caused because one or the other had modified the end of the same structure or macro, but luckily it turned out nothing significant and I was able to easily reconcile things. The hardest was because OpenWRT modifies vmstat.c for MIPS and the same code was modified by grsecurity to add extra memory protections.  At this point I attempted to build the system, and discovered three other minor cases that broke the build. These mispatches essentially were due to movements in one or two lines, or new code using internal kernel API modified by grsecurity, and were also easily repaired.  The most difficult mispatch to understand was where OpenWRT rewrites the kernel module loader code, apparently to make better use of MIPS memory structures and it took me a little while to understand how to try and fix things.

The end result is on github at [https://github.com/pastcompute/openwrt-cc-linux-3.14.x-grsecurity](https://github.com/pastcompute/openwrt-cc-linux-3.14.x-grsecurity)


## Applying grsecurity - OpenWRT quirks


One strange bug that had to be worked around was some new dependency in the kernel build process, where extra tools that grsecurity adds were not being built in the correct order with other kernel prerequisites.

In the end I had to patch how OpenWRT builds the kernel to perform an extra '_make olddefconfig_' to sort things out.

I also had to run '_make kernel_menuconfig_' and turn on grsecurity.

As the system built, I eventually hit another problem area: building packages. This was a bit of an 'OH-NO' moment as I thought it had the potential to become a big rabbit hole. Luckily as it turned out, only one package was affected in the end: compat-wireless.  This package builds some extra user space tools and wifi drivers, and used a macro, _ACCESS_ONCE_, that was changed by grsecurity to be more secure; and required use of a new macro to make everything work again, _ACCESS_ONE_RW_. There were rather a number of calls to this macro, but luckily it turned out to be fixable using sed!


## Booting OpenWRT with grsecurity - modules not loading


I was able to then complete an INITRAMFS image that I TFTP'd into my carambola2 via uboot.

Amazingly the system booted and provided me with a prompt.

[crayon lang="plain"]
U-Boot 1.1.4-g33f82657-dirty (Sep 16 2013 - 16:09:28)

=====================================
CARAMBOLA2 v1.0 (AR9331) U-boot



Starting kernel ...

[ 0.000000] Linux version 3.14.26-grsec (andrew@atlantis4) (gcc version 4.8.3 (OpenWrt/Linaro GCC 4.8-2014.04 r43591) ) #3 Sun Dec 14 18:08:52 ACDT 2014
[/crayon]

I then discovered that no kernel modules were loading. A bit of digging and it turns out that a grsecurity option, _CONFIG_GRKERNSEC_RANDSTRUCT_  will auto-enable _CONFIG_MODVERSIONS_. One thing I learned at this point is that OpenWRT does not support _CONFIG_MODVERSIONS=y_, due to the way it packages modules with its packaging system. So an iteration later with the setting disabled, and everything appeared to be "working"


## Testing OpenWRT with grsecurity


Of course, all this work is moot if we cant prove it works.

Easy to check is auditing. For example, we now had these messages:

[crayon lang="plain"]
[ 4.020833] grsec: mount of proc to /proc by /sbin/init[init:1] uid/euid:0/0 gid/egid:0/0, parent /[swapper:0] uid/euid:0/0 gid/egid:0/0
[ 4.020833] grsec: mount of sysfs to /sys by /sbin/init[init:1] uid/euid:0/0 gid/egid:0/0, parent /[swapper:0] uid/euid:0/0 gid/egid:0/0
[ 4.041666] grsec: mount of tmpfs to /dev by /sbin/init[init:1] uid/euid:0/0 gid/egid:0/0, parent /[swapper:0] uid/euid:0/0 gid/egid:0/0
[/crayon]

However, the acid test would be enforcement of the NX flag. Here I used the code from [http://wiki.gentoo.org/wiki/Hardened/PaX_Quickstart](http://wiki.gentoo.org/wiki/Hardened/PaX_Quickstart) to test incorrect memory protections. Result:

[crayon lang="plain"]

[19111.666360] grsec: denied RWX mmap of <anonymous mapping> by /tmp/bad[bad:1497] uid/euid:0/0 gid/egid:0/0, parent /bin/busybox[ash:467] uid/euid:0/0 gid/egid:0/0
mmap failed: Operation not permitted

[/crayon]

Success!


## Revisiting Checksec, and tweaking PAX


In an earlier blog I wrote about experimenting with checksec.  Here I used it to double-check that the binaries were built with NX protection. MOst were, due to a patch I previously submitted to OpenWRT for MIPS. However, openssl was missing NX. It turns out that OpenSSL amongst everything else it has been discussed for this year, uses assembler in parts of the encryption code! I was able to fix this by adding the relevant linker '_.note.GNU-stack_' directive.

The PAX component can be tweaked using the paxctl command, so I had to build that with the OpenWRT toolchain to try it out. I discovered that it doesnt work for files on the JFFS2 partition, only in the ramdisk. Further to enable soft mode, you need to add a kernel boot command line argument. To do this for OpenWRT, edit a file called _target/linux/$KERNEL_PLATFORM/generic/config-default_ where in my case, _$KERNEL_PLATFORM_ is ar71xx


## Moving Targets


Right in the middle of all this, OpenWRT bumped the kernel to 3.14.26. So I had to exercise a workflow in keeping the patch current.  As it happened the grsecuroty patch was also updated to 3.14.26 so I presume this made life easier.

After downloading the stock kernel and pulling the latest OpenWRT, I again re-created the patch series, then applied grsecurity 3.14.26.  The same four rejects were present again, so fingers crossed I cherry-picked all my work from 3.14.25 onto 3.14.26. As luck would have it this was one smooth rebase!


## Recap of OpenWRT grsecurity caveats





	
  * _CONFIG_GRKERNSEC_RANDSTRUCT_ is not compatible with the OpenWRT build system; using it will prevent modules loading

	
  * Some packages may need to be modified to support NX - generally, if these use assembly language and don't use the proper linker directive.

	
  * For some reason paxctl only seems to work on files in _/tmp_ not in the JFFS overlay. This is probably only a problem when debugging

	
  * Your experience with the debugger gdb will probably be sub-optimal unless you put the debug target on /tmp and use _paxctl_ to mark it with exceptions




## Summary


After concluding the above, I converted the change set from my local Linux working copy into a set of additional patches on OpenWRT and rebuilt everything to double check.

The branch 'ar71xx-3.14.26-grsecurity' in [https://github.com/pastcompute/openwrt-cc-ar71xx-hardened](https://github.com/pastcompute/openwrt-cc-ar71xx-hardened) has all the work, along with some extra minor fixes I made to some other packages related to checksec scan results.

**THIS MAY EXPLODE YOUR COMPUTER AND GET YOU POWNED! This has been working for me on one device with minimal testing and is just a proof of concept.**
