---
author: admin1
comments: false
date: 2015-09-04 11:25:52+00:00
layout: post
slug: distracting-adventures-in-zfs-upgrades
title: Distracting adventures in ZFS upgrades
wordpress_id: 671
categories:
- linux
tags:
- boot
- embedded
- grub
- kernel
- zfs
---

Last week I wanted to play around with some software packages for logging and charting of environmental measurements and events (specifically, two packages, [openhab](http://www.openhab.org), and [emoncms](http://emoncms.org))
Wanting to save time (sweet irony!), rather than building up a VM and manually configuring the tools, I figured I'd use docker. Except that the workstation I wanted to use was running Debian Squeeze was still on kernel 3.2, which doesn't support docker. Oh, and a ZfsOnLinux (ZoL) zraid for the root filesystem.
So the steps to get to docker involved upgrading the kernel, ZFS, and by the way, the nvidia drivers.
_**Mistake #1. I should have just built a Xubuntu 14.04 VM and run docker inside that!**_

Before upgrading the kernel from Debian [backports](backports.debian.org), I decided to ensure ZfsOnLinux was updated. I (correctly, confirmed) anticipated the most problems with ZoL. Anyway, I knew that upgrading ZoL would be fraught with danger so I read all the documentation, and upgrade advice, and so on, and took all the recommended precautions.

But of course, after going through two cycles of _apt-get_ and _dpkg-reconfigure_ and  rebuilding the initramfs and so on, after rebooting, BAM! A variant of the dreaded "failed to mount the root filesystem" error. Reported close by was a missing kernel module error for something called zcommon.

After a bit of digging and breaking the virtual glass on an emergency boot partition I worked out that I had missed upgrading one of the packages required for ZoL. Why it was not an automatic dependency I don't know, but after installing something called "_libvnpair_" the system booted further. And then stopped again.

This one would take rather a bit more work to track down. Semi-helpfully, the entire error message was:

    
    Command:
    Message:
    Error:
    Manually import the root pool at the command prompt and then exit.
    Hint: Try: zpool import -R / -N ${ZFS_RPOOL}


At this point, the initramfs was dropping my system to a rescue shell, and via the above message advising me to import the ZFS pool containing the root filesystem. So I tried its helpful suggestion to execute the 'zpool import' command, which actually succeeded, and after some more fiddling manually mount various file systems proceeded to boot the system. However, this manual process only got me out of trouble once, and still needed to be resolved.

To get further I had to instrument the initramfs file _scripts/zfs_ with a bunch of echo statements and rebuild the initramfs. (The script files bundled in when rebuilding initramfs on Debian are located under _/usr/share/initramfs-tools/scripts_) This let me reboot and work out where the zpool import was failing (or not even being called at all.)

As it turns out, _zpool_ was not being called, at all, in a way that would work for my partitioning scheme. The logic in _scripts/zfs_ runs a whole bunch of permutations trying to locate the pool, but if a variable called _ROOT_ is empty it skips executing _zpool_ as required. The solution, as it turns out, was to update my grub with '_**root=zfs:AUTO**_' - previously, my kernel did not require this kernel argument, but now, having upgraded ZoL, from 0.6.2 to 0.6.4, it did.

So, what caused this? There were a lot of year or so old threads discussing upgrade errors related to ZfsOnLinux but none of them quite matched my specific scenario.

One possibility is this:
* I run a separate boot filesystem from the usual /boot, containing a hand crafted grub, which can execute various tools such as Gparted, various minimal linux installs for rescue purposes, memcheckx86 and other tools.
* Whenever I upgrade the kernel on this system I need to copy over the _vmlinux_ and _initramfs_ files to this originating boot filesystem from /boot (which is never used by my grub)
* I wonder if ZoL  may have added the _**root=zfs:AUTO**_ option to the Debian grub update facility, but I neglected to check for changes to the generated _/boot/grub/grub.cfg_ and apply any changes to  my real _grub.cfg_. And wham!

However, I couldn't find any references to zfs in _/etc/grub.d_, so this hypothesis may well be wrong. Via occams razor, perhaps its just that my setup on this particular workstation is more complex or unusual than most users of ZfsOnLinux. Anyway, onward and upwards.

I' shortly to decide on which of OpenHAB or EmonCMS I'll be using for my [Hackaday Prize finals entry](https://hackaday.io/project/4758). Stay tuned!
