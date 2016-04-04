---
author: admin1
comments: false
date: 2013-01-28 13:27:59+00:00
layout: post
slug: aspireone-and-encrypted-sd-card-home
title: AspireOne and encrypted SD card /home
wordpress_id: 393
categories:
- linux
tags:
- boot
- cryptsetup
- power
---

Tip for the week (well I finally sorted this just in time on Sunday for LCA2013 )

If you run /home with encryption it doesn't come back properly after suspend resumes, unless you add the following to your kernel boot command line:

mmc_core.removable=0

I think this means the hot removal of the SD card slots doesn't work so dynamically but in my case, it means I can suspend my machine - which I need to do a lot at LCA.

(With thanks to [https://bbs.archlinux.org/viewtopic.php?id=91807](https://bbs.archlinux.org/viewtopic.php?id=91807) )

I found this out after rebuilding my trustty aspireone to Crunchbang Waldorf and suddenly suspend appeared to cause all manner of problems.
