---
author: admin1
comments: false
date: 2013-08-30 09:22:45+00:00
layout: post
slug: something-new-dcfldd-a-more-advanced-dd-for-data-transfer
title: 'Something new: dcfldd - a more advanced dd for data transfer'
wordpress_id: 441
categories:
- howto
tags:
- linux
- something new
---

Today I discovered [dcfldd](http://dcfldd.sourceforge.net/), and it was right there in Debian already: `apt-get install dcfldd`.

This was while [building](http://elinux.org/RPi_Easy_SD_Card_Setup#Using_the_Linux_command_line) up a new Raspberry Pi image for a project I have in mind.

Specifically, dcfldd provides a progress meter, unlike the ubiquitous dd command.  It also appears to be aimed at forensics and advanced data recovery, featuring on the go hashing of data as well.

You can use dcfldd to write an image onto an sdcard in exactly the same way as dd:


    
    dcfldd bs=4096 if=2013-07-26-wheezy-raspbian.img of=/dev/sdl



Sample Progress Output:

    
    141568 blocks (553Mb) written.



It looks like you can get a Windows binary as well.
