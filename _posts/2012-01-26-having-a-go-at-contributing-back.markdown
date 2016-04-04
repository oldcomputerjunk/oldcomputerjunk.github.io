---
author: admin1
comments: true
date: 2012-01-26 12:11:06+00:00
layout: post
slug: having-a-go-at-contributing-back
title: Having a go at contributing back
wordpress_id: 225
categories:
- linux
tags:
- debian
- patch
---

Feeling somewhat inspired by LCA2012 I have a large list of possible things to try, although I am mindful of not committing to too many projects as was pointed out by Rusty in the newcomer session (which I missed but managed to catch on YouTube after getting home)

Then today whilst performing a diagnostic on my home wifi I actually hit a bug, and this time found myself in a position to not only report the problem but actually submit a patch for the first time ever!

Using a tool called [bing](http://packages.debian.org/squeeze/bing) to try and measure bandwidth I noticed weird output when for kicks I pointed it at a site on the Internet.  Further digging uncovered a bug filed against the package in the Debian bug system.  To cut a long story short I worked out the problem (ICMP message timeouts do not always get reported properly) and made a short patch and submitted this by email to the bug tracker.

Actually I sent that 15 minutes ago but haven't yet seen it on the web page for the bug, or received any automated reply, so I am currently wondering if my email got through... I guess I have a learning curve ahead of me



### Update

my bug report and patch is up at [bugs.debian.org/cgi-bin/bugreport.cgi?bug=464257](http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=464257), woot!
