---
author: admin1
comments: false
date: 2012-11-11 05:50:24+00:00
layout: post
slug: windows-shenanigans-with-a-little-help-from-our-linux-friends-part-1
title: Windows shenanigans - with a little help from our (Linux) friends - part 1
wordpress_id: 356
categories:
- realworld
- tech
- windows
tags:
- boot
- disks
- microsoft
- vista
---

Although I use Linux as my primary O/S, I am required to also use Windows at work and most family / friends / neighbours etc. use it.  So I need to stay up to date with the Microsoft world to retain my computer geek "cred", as I am often called upon to fix problems or provide tuition...

Quick tip if you have to use a Microsoft O/S - you may be able to resolve Windows Vista / Windows 7 boot problems using [EasyBCD](http://neosmart.net/EasyBCD/), it is free for non-commercial use.  Similar can be accomplished using Grub2 and GPartEd; however EasyBCD can  manipulate the native Windows boot manager, and I need to experiment further with my wifes Win7 laptop when she is not around ;-)

A while ago a close relative had a run of bad luck with his system.  Amongst other things this involved migrating from an old "slow" Vista Premium to a fresh install, on a clean hard drive.  The fresh install ran much faster without the years of crud build up and recent drivers, etc. but he was unable to make it work without the original "slow Vista" hard disk in the machine, which was the system (BIOS) boot disk.  The computer involved had several internal SATA and external drives, a situation which had previously eventually lead to disaster as my relative attempted to sort it all out, but more on that another time!

The problem was the "new" Vista was added to the Windows boot menu but with the "old" drive removed, the system was rendered unbootable; i.e. the clean drive had no boot manager installed.

The solution I employed was to use a tool called [EasyBCD](http://neosmart.net/EasyBCD/).  The procedure essentially involved first installing EasyBCD onto the old Vista, and using it to make the new drive the default, at which point we went out to lunch at least making his system slightly more usable.
Having confirmed the new Vista was automatically entered on reboot, EasyBCD was installed into the "new" Vista, and used to install the boot manager onto the new drive, and finally removing the old drive.  One key step involved using the "Select BCD store" to edit the menu on the alternative disk.
In all cases, it is prudent to take a BCD backup! (And of course backup anything else important.)

This was not completed without some recourse to Linux; at the start of proceedings, in spite of a lot of to-ing and fro-ing of drives and cables, neither Vista system would recognise a new 2TB drive he wished to use for data.  I was able to boot using Xubuntu 12.04 and this could not properly see the drive either!  As a last resort we swapped it to a USB a caddy and using my Acer Aspire One running Squeeze with a 3.2.9 kernel confirmed the drive was OK.  Then running a manual Windows update on Vista actually allowed the system to recognise the drive.  Perhaps I should have done this first, but I think it can be useful to experiment a bit longer and it was more comfortable inside on this day anyway... S

It seems therefore that both older unpatched Microsoft systems and older Linux kernels cant see some larger hard disks.

This all happened a little while ago so I don't have exact model numbers or software versions.
