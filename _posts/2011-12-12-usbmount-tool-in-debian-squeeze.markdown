---
author: admin1
comments: true
date: 2011-12-12 03:08:11+00:00
layout: post
slug: usbmount-tool-in-debian-squeeze
title: USBmount tool in Debian Squeeze
wordpress_id: 34
categories:
- howto
- linux
tags:
- debian
- disks
- linux
- usb
---

For reasons which escape me at the moment I decided to ditch the default USB auto-mounting that is in the Gnome in Debian Squeeze and try out various alternatives. I ended up using the [usbmount](http://packages.debian.org/squeeze/usbmount) package which I installed some months ago.
This package did the job dutifully placing some usb0 or usb1 icon on my desktop when I plug in my USB drives, with some niggles that got more annoying over time... <!-- more --> Life being what it is I managed to put up with an annoying problem where only root (or via sudo) could write to the disk for quite some time.   This became extra problematic when trying to encourage my 10 year old to use this computer to email photos to family members...
Today I finally fixed this by editing the file `/etc/usbmount/usbmount.conf` and setting NTFS and FAT partitions to be permissioned with `nobody,users` ... maybe not the 'best' practice but more than sufficient for my main desktop which is only used by me, my son and occasionally my SO (when her Windows7 laptop breaks:-) )

To configure usbmount to mount USB drives writable by everyone in the users group, simply append the following to `/etc/usbmount/usbmount.conf`.  This will make it all magically work, no services even needed restarting:
[crayon lang="bash"]
MOUNTOPTIONS="sync,noexec,nodev,noatime,nodiratime"
FS_MOUNTOPTIONS="-fstype=vfat,gid=users,uid=nobody,umask=002,sync \
                 -fstype=ntfs,gid=users,uid=nobody,umask=002,sync"
[/crayon]

The `atime`, etc. options are for performance and reducing writes onto Flash.  The file seems to be shell sourced somewhere inside udev so normal shell scripting could be used if desired.

