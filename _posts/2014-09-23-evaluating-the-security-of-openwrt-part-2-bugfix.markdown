---
author: admin1
comments: false
date: 2014-09-23 11:59:24+00:00
layout: post
slug: evaluating-the-security-of-openwrt-part-2-bugfix
title: Evaluating the security of OpenWRT (part 2) - bugfix
wordpress_id: 578
categories:
- blog
tags:
- blog
- openwrt
- security
---

I had a bug applying the RELRO flag to busybox, this is fixed in GitHub now.

For some reason the build links the busybox binary a second time and I missed the flag.

Also an omission from my prior blog entry: uClibc has RELRO turned on in its configuration already in OpenWRT, so does not need flags passing through to it. However, it is failing to build its libraries with RELRO in all cases, in spite of the flag.  This problem doesn't happen in a standalone uClibc build from the latest uClibc trunk, but I haven't scoped how to get uClibc trunk into OpenWRT.  This may have been unclear they way I described it.
