---
author: admin1
comments: false
date: 2014-08-23 09:47:38+00:00
layout: post
slug: raspberry-pi-virtual-machine-automation
title: Raspberry Pi Virtual Machine Automation
wordpress_id: 545
categories:
- howto
- linux
tags:
- embedded
- linux
- pi
- qemu
- virtualisation
---

Several months ago now I was doing some development for Raspberry Pi. I guess that shows how busy with life things have been. (I have a backlog of lots of things I would like to blog about that didn't get blogged yet, pardon my grammar!)

Now the Pi runs on an SD card and it some things start to get very tedious after a while, write performance is not exactly fast, and doing extensive work would probably start to wear cards a bit fast.

So I looked into running a Raspberry Pi Qemu virtual machine, which would let me do builds on my main workstation without needing to setup a cross compiling buildroot. This has the further advantage in that I could test automating full installations for real Raspberry Pis.


## Overview


Because I really dislike having to lots of manual steps I automated the whole process. The scripts I used are on [GitHub (https://github.com/pastcompute/pi_magic)](https://github.com/pastcompute/pi_magic), use at your own risk etc.

There are three scripts: `pi_qemu_build.sh` which generates a pretend SD card image suitable for use by Qemu; `pi_qemu_setup.sh` which will SSH into a fresh image and perform second stage customisation; and `pi_qemu_run.sh` which launches the actual Qemu VM.

My work is based on that described at [http://xecdesign.com/qemu-emulating-raspberry-pi-the-easy-way/](http://xecdesign.com/qemu-emulating-raspberry-pi-the-easy-way/).


## Image Generation


The image generation works as follows:



	
  * start with an unzipped Raspbian SD card image downloaded from [http://www.raspbian.org/](http://www.raspbian.org/)

	
  * convert the image to a Qemu qcow2 image

	
  * extend the size out to 4GB to match the SD card

	
  * mount using Qemu NBD and extend the ext2 partition to the rest of the disk

	
  * Patch the image to work with Qemu quirks (more on that in a moment)

	
  * Create a snapshot so that changes made later inside Qemu can be rolled back if required


Now Raspbian is setup to load a DLL `/usr/lib/arm-linux-gnueabihf/libcofi_rpi.so`, which needs to be disabled for Qemu; also it is set up to search for an MMC device so we need to make a symlink to access `/dev/mmc*` as `/dev/sda` inside Qemu instead.


## Qemu Execution


The stock Raspberry Pi kernel wont work inside Qemu, so we need to launch with a different kernel.

I haven't yet had time to include producing one in the build process.  Instead, I launch Qemu using the kernel downloadable from [XEC Design](http://xecdesign.com/qemu-emulating-raspberry-pi-the-easy-way/).

Qemu is invoked using `qemu-system-arm -kernel path/to/kernel -cpu arm1176 -m 256 -M versatilepb ...`, so that the processor matches that used by the Raspberry Pi. The script also maps HTTP and SSH through to the VM so you can connect in from localhost, and runs using the snapshot.

This was tested on Debian Wheezy with qemu 1.7 from backports.

[caption id="attachment_548" align="aligncenter" width="300"][![PI VM screenshot](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/08/pivm-300x233.png)](http://blog.oldcomputerjunk.net/wp-content/uploads/2014/08/pivm.png) PI VM screenshot[/caption]
