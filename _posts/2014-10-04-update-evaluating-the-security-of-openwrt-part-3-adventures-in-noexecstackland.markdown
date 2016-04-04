---
author: admin1
comments: false
date: 2014-10-04 00:19:28+00:00
layout: post
slug: update-evaluating-the-security-of-openwrt-part-3-adventures-in-noexecstackland
title: (UPDATE) Evaluating the security of OpenWRT (part 3) adventures in NOEXECSTACK’land
wordpress_id: 585
categories:
- infosec
- linux
tags:
- embedded
- linux
- openwrt
- security
---

Of course, there are more things I had known but not fully internalised yet. Of course.

Many  MIPS architectures, and specifically, most common router architectures, don't have hardware support for NX.



Yet. It surely wont be long.



My own feeling in this age of Heartbleed and Shellshock is we should get ahead of the curve if we can - if a distribution supports NX in the toolchain then when a newer SOC arrives there is one less thing that we can forget about.

If I had bothered to keep reading instead of hacking I may have rediscovered sooner. But then I would know significantly less about embedded toolchains and how to debug and patch them.  Anyway, a determined user could also cherry-pick emulated NX protection from PAX.
When they Google this problem they will at least find my work.



How else to  learn?  :-)
