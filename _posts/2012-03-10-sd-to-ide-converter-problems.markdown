---
author: admin1
comments: false
date: 2012-03-10 13:12:12+00:00
layout: post
slug: sd-to-ide-converter-problems
title: SD to IDE converter problems
wordpress_id: 277
categories:
- tech
tags:
- ebay
- hardware
---

I recently bought an SD to IDE bracket converter off ebay.  It only cost me $11 from one of those cheap Chinese stores with feedback in the 10s of thousands so if it didn't work I hadn't lost much money at least.  It looks very much like [this](http://www.soarland.com/3-LEDs_Bracket_40-Pin_IDE_To_SD_Card_Adapter-product-252.html).
I have had good success using Compact Flash as a bootable SSD for my old 386 and 486 boxen so given the wider availability of cheap SD cards (on the shelf at the local supermarket for $7) I decided to try an SDcard version - currently I have an old Pentium4 I am rebuilding for use as a hardware test machine in the shed.  Having a card running FreeDOS and a Linux distro with a shared FAT partition is handy for transferring between different PCs and even to my main computer when network is unavailable.
After much mucking around it seemed I had a DOA (incidentally in spite of what you might expect I have had very few DOA in >20 cheap Ebay purchases) as whenever I had it connected to the IDE bus with a card inserted no devices were detected; tested and failed with two different 8GB SDHC cards.  I wasn't hopeful as the soldering on the board was terrible, and reworking surface mount solder without proper equipment is painful (having done so with good results on a dodgy ebay purchase before)
In the end for some reason I tried it with a 1GB card from my spare camera and the motherboard detected the reader as an IDE device.  Further testing confirmed this gadget seems to not cope with 8GB SD cards.

I already had another one of these gadgets but without the PC case bracket, that works with 8GB cards, so this one must have some firmware variation that limits it to 4GB.

I messaged the seller querying about possible firmware upgrade but I don't hold much hope for that.

So in summary, if you buy one of these, be prepared for some issues with various SD cards.
