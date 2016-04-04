---
author: admin1
comments: false
date: 2013-02-28 09:02:46+00:00
layout: post
slug: quick-tip-legacy-skype-on-debian-squeeze
title: Quick Tip - Legacy Skype on Debian Squeeze
wordpress_id: 409
categories:
- howto
- linux
tags:
- debian
- skype
---

The trick to making the webcam work:

    
     LD_PRELOAD=/usr/lib32/libv4l/v4l1compat.so /usr/bin/skype


This assumes of course you have the various ia32- and lib32- packages installed.

This is with a skype 2.2.0.35 deb I made a couple of computer builds ago, so YMMV.

With thanks to: [http://community.linuxmint.com/tutorial/view/219](http://community.linuxmint.com/tutorial/view/219)


