---
author: admin1
comments: false
date: 2013-01-28 13:03:26+00:00
layout: post
slug: linux-conf-au-2013-day-1
title: Linux.conf.au 2013 Day 1
wordpress_id: 389
categories:
- linux
tags:
- LCA2013
---

## Thoughts on Linux.conf.au, Canberra: Day 1, January 28th


Well I am taking a quick break from tweaking my open programming  miniconf talk in preparation for tomorrow. I have only done a handful of public talks and only one in front of an audience that might be around the same size so the butterflies are fluttering just a tiny bit.

One thing I have been doing is watching some of the other other talks to watch presentation techniques and be inspired, and hopefully I can remember to put some of it into practice.  It seems the bar set at LCA is always a high one!

I really enjoyed the keynote by Bdale Garbee. It covered issues such as corporate involvement with open source (companies do need to make money, and this is ok), increasing complexity in open source restricting opportunity for small contributions, and the future of Linux desktop and evolution of the desktop.  I found quite a lot of his points striking a personal chord, having in recent times running into various roadblocks resulting in 2am oft-fruitless hacking sessions trying to make software "work" - not just how I want, but in a way I thought might have been, if not obvious to any software developer, at least possible given all the software that has existed in the past ... it gets frustrating when software just seems to go backwards (or maybe its just my lawn) - but to quote Bdale , **"we should write the software we want"** where we is the  developer, and here I interpret this (hopefully correctly) to mean to not exclude "ourselves" as "hackers" just to make life easier for "users"

Of the miniconfs, my friend Kim Hawtin had a good presentation at the sysadmin miniconf today, I dont envy having to deal with that many web servers; Kevin Pulo with his live demo of Syzix showed how to pull a pull off a talk given constrained timeframe and Sheeri Cabrals advice for dealing with the filler "umms" in her captivating talk on podcasting of course translates to public speaking (hope I manage to avoid those!)

I wont be doing any live demos though...

Speaking of live demos, the day finished with the FirefoxOS. Redundancy saved the day here, the Mozilla presenters had two devices - this paid off when the first did not make a network connection; he was able to step in with a successful run of the Hacker beach app demo.  It is interesting they can make this work on low-spec hardware, if it can be ported to many older devices this may ever so slightly breathe live into otherwise good mobile equipment that might go onto landfill, serving as spare or even main phones.


## (Editorial Follows - skip if TL;DR )


Here I will disclose, although I spend many of my days in front of Linux, using open source tools, and developing with open source libraries, apart from this blog, intermittent bug reports and some very minor patch submissions and failed starts**  I have not (yet!) managed to contribute much in the way of code to open projects.  Although I am considering strategies to increase my contributions,  sadly most of my career I have been working the wrong side of a virtual or physical firewall of some kind.  However I am hacker culture at heart, having a lifelong passion for all things computer, and those who know me are certainly aware of my enthusiasm for open source...  so I am going to be inspired from Paul Fenwicks keynote last year and bury my "imposter" feelings, and actually comment on the following issue anyway.  One thing I certainly learned at my first LCA last year, is you may as well participate as much as possible, as there are few opportunities in life like this one!

Regarding some of the recent developments in desktop software, alluded to by Bdale in the keynote.  My own personal example: udisks. I recently had to do some maintenance on a box running a mixed debian/ubuntu "franken"-distro.

Well, you get what you ask for maybe.

However, all I wanted to do is insert a USB stick and have it mount with the filesystem options I want.  On this system this was accomplished by a thing called UDisks, which I later found out seems to be part of some grand FreeDesktop.org software conglomeration.  It turns out that the mount options are _hard coded_ in the source. What? Really?

I have been in the software dev business for many years now, and one thing I drum into the grads I sometimes have to mentor is the usefulness of not hard coding _everything_ in any kind of non-simple software.  It just makes like too hard for diagnostics developing and tweaking, especially when you have layers of packaging and deployment in the way.  Sure you need some sane defaults, and you don't want to go overboard with configuration because it can introduce maintenance complexity.  But I would have thought something as core as mount options, at least, would be configurable.  Not according to at least one UDisks bug resolver.  There is a bug report*** filed against UDisks asking for just this feature, and the attitude of the triager appeared to be "why would you want this?", and in spite of use cases provided, the issue appears to have been dismissed out of hand.

(There are plenty of workarounds; using usbmount as per my earlier [blog](http://blog.oldcomputerjunk.net/2011/usbmount-tool-in-debian-squeeze/) is one; but here UDisks serves to illustrate my point about making software unconfigurable.  And you cant always control what is installed on a machine you are working on.)

Other forums had patches of the C source code to do what you want; sure open source means you can do this, but why make things hard for ourselves?

This seems to be an instance of the increasing complexity in contributing to projects Bdale discussed in the keynote.

I would make the observation this all seems to be part of a broader split between 'free desktop' on one hand and 'the unix way' on the other, best summed up in the following blog, which I found whilst researching my travails with UDisks : Linux Futures, [http://www.pappp.net/?p=969](http://www.pappp.net/?p=969)

All that aside, I am now running CrunchBang on my AspireOne and it works brilliantly.  OpenBoxe + XFCE + Debian FTW!



Must finish miniconf talk now...



** I ported to a Sane backend a driver for a Canon FS4000  slide scanner from code originally produced for Windows, but only got it working to what I would consider alpha stage; the Sane project devs consider it too immature for inclusion into the main line, as they rightfully should, and then I had to return the scanner to its owner before being able to finish it.  I loaded the lot onto github ( [https://github.com/andymc73/sane-fs4000-backend](https://github.com/andymc73/sane-fs4000-backend) ) in case it proves useful to someone else

*** I cant put my hands on the report atm., it is in my browser history, at home not here on my trusty netbook! and my google-fu is not quite working for me just now)
