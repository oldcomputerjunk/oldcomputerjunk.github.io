---
author: admin1
comments: true
date: 2011-12-27 06:46:10+00:00
layout: post
slug: debian-squeeze-and-linux-3-the-saga-begins
title: Debian Squeeze and Linux Kernel 3
wordpress_id: 52
categories:
- linux
tags:
- debian
- kernel
- linux
---

A couple of months ago I splashed out and upgraded my motherboard to an ASUS [Sabretooth 990FX](http://www.asus.com/Motherboards/AMD_AM3Plus/SABERTOOTH_990FX/), in theory to complement my quad core Athlon II 640 with faster RAM and SATA 3, more PCI-E slots etc. (but the camoflauge colour scheme and mil-spec capacitors didn't hurt either... `:-)` ) 

However it wasn't all smooth sailing, <!-- more -->and my system just hasn't been as stable as it was with my previous board, a Gigabyte [GA-MA78S2H](http://www.gigabyte.com/products/product-page.aspx?pid=2926#ov).  With my new setup, I have had issues with television (DVB-T) no longer working after waking from sleep, surround sound not working properly, USB devices not working properly after wake-up, and random non-fatal kernel stack dumps.

Over the Christmas break I started to try and sort it all out.

So to start with I figured I'd try a newer kernel - I was currently running a 2.6.39 series from the Debian [](http://backports-master.debian.org/) archive.

It turned out that I was able to install a kernel from the development repository (aka 'wheezy') without it wanting to pull in a heap of dependencies.  This is good because I like to try and track stable as much as possible, because I keep DVD images locally to avoid reduce bandwidth usage when installing new packages.  _But all is not as it seems... (see update below)_

Reboot, OK so far so good.  But first hurdle was when GDM would have  started, the display remained stuck on a black screen with a fast flashing text cursor.  Rebooting into single user mode, I confirmed that X was failing to start.  

Of course, after some brief checks it was because I was using binary NVIDIA drivers and they had not recompiled for my new kernel, in spite of my DKMS setup.

To get moving I switched X to use the nouveau driver.  To do this I  edited `/etc/modules` and added the nouveau module, and moved `/etc/X11/xorg.conf` out of the way. This at least got the system going again.




### Update


It turns out the DKMS driver rebuild failed because I had not installed the kernel headers.  And upon attempting to do so, of course aptitude wanted to pull in gcc-4.6 and many other dependencies from wheezy, which I do not want to deal with yet.
So revert back to 2.6.39 for the time being...
