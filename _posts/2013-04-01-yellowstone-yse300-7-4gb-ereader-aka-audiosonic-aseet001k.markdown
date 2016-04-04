---
author: admin1
comments: false
date: 2013-04-01 11:54:25+00:00
layout: post
slug: yellowstone-yse300-7-4gb-ereader-aka-audiosonic-aseet001k
title: Yellowstone YSE300 7" 4GB EReader (aka AudioSonic ASEET001K)
wordpress_id: 414
categories:
- tech
tags:
- ereader
- hacking
- hardware
---

(This is a late night blog entry 'get it down before it gets forgotten', mainly in case it is of use to someone, so please excuse the untidyness!)

I had noticed these for some time at JB HiFi for $59 but ignored them, being not touch screen and
not wanting to waste the money for what seemed to be an underwhelming device I didnt need.
However I was browsing the other day and they were 20% off that, so in a 'what the hell' moment
I purchased one to experiment with.


## First Impressions


The feature list on paper seems reasonable, for some definition of reasonable. At least, compared to what the average Joe would be paying at least $100 for to a walk out the shop with an android device or a Kobo, etc.

The device is TFT LCD 800x480 resolution in 15:9 widescreen with headphone socket, micro USB for charging / connectivity, and is not touch screen, but button driven.

It runs a custom firmware, with MP3 player, video player, calendar, calculator, voice recorder,
and e-reader.  I have only tested the e-reader using PDF, and the photo viewer and MP# player.
The device is recommended to work with the Adobe e-reader Windows software and support DRM'd
ebooks but I didnt test with that.  It has a TF (mini-sdcard) slot.



## In Use



In practice when reading a PDF I found it to be significantly sluggish at
rendering the pages.  The rendering quality of the fonts found me struggling to read late
at night as well.  The test book I used was the free download edition
of "Hacking the X/Box" ([http://boingboing.net/2013/03/11/hacking-the-xbox-free-in-hono.html](http://boingboing.net/2013/03/11/hacking-the-xbox-free-in-hono.html))  by Bunnie Huang,  who was a keynote speaker at LCA2013.
He released for free as a tribute to Aaron Swartz, and I found it quite the suitable choice to
read whilst attempting to work out if this device was itself hackable in any way.

Pages with pictures could take several seconds to render, which is something I havent
noticed when using the no-brand $65 Android tablet I purchased from ebay for my kids to use.

On the upside, it competently does photo-frame, and the MP3 playback with a choice of equalisers.
It also supports voice record, video play, FM radio calculator and calendar, I didnt bother trying these.
So the list of upsides is quite short.

The UI however is average, and the physical interface downright woeful.  There is a lot of lag responding to button presses.  The poor quality is most noticeable with MP3, all the buttons seemed to just seek tracks and I couldn't work out how to control the volume!  According to the manual, 'up' and 'down' (as one might expect)
should control the volume, but it is difficult to do just this without pressing the central
(ok)  button at the same time unless you are very careful.
And the usual response to the wrong button is to restart the current song...

The player played music without headphones but at a very low volume.



## Can it be hacked?


The intent of my experiment was to try an alternative firmware, such as found at sites like [rockbox.org](http://rockbox.org).

Inside, there are three chips, an RK2738 SOC (ARM), and a FLASH and a RAM.  However, it seems the RK2738 is a somewhat underwhelming device from a hackability perspective: initial google searching returned a dearth of information.

The Yellowstone / Audiosonic has no firmware tool on an website relatad to either JB or Kmart or either search term.
I did find a RK27 SDK I could download, and multitude  of firmware loaders.  But none worked!
In theory the device can be switched into a service mode mode where system partitions become accessible, this would allow the ROM to be dumped, etc.
But none of the suggested  methods worked.
First method - write a zero length 'tag' file in the root directory, and reboot.  I tried a variety of
combinations seen on various forums for similar devices, to no avail. (Example: http://forums.rockbox.org/index.php?topic=34081.0 )
Second method - try the firmware tools of various other devices based on the RK2738, but the tools that actually detected the device as a RockChip require it to be in the system mode.

Going by the SDK and some forum posts, it should be possible to switch to system mode by sending
a specially crafted SCSI command (being a mass storage device, SCSI protocol is used over USB.)
But this approach also failed.

Further research on the RK2738 hinted as to whether rockbox (or even some Linux) might work, but
I can't experiment until I can get it into system mode. It seems the RK2808 or upwards is needed for Android, so I might be left with RockBox:  [http://www.rockbox.org/wiki/Rockchip27xxPort](http://www.rockbox.org/wiki/Rockchip27xxPort)

The only thing I have left to try, as 'reboot' has some ambiguity, is to disconnect the battery and
do a true cold reboot, to see if any of the tag files might have an effect.
Failing this, you can short pins on the ROM.  Life however leaves me no time to waste on the above, so this has now turned into yet another rainy day project.

Of note, the device looks identical to the Audiosonic Ereader marketed by Kmart. ([http://www.e3style.com/general/eReaders/Audiosonic/ASEET001K-UM_V4_24Aug2012.pdf](http://www.e3style.com/general/eReaders/Audiosonic/ASEET001K-UM_V4_24Aug2012.pdf))

Otherwise, it will have to do duty as a e-reader (until I get sick of the rendering)
media player ( for which I already have my phone for mp3) and higher resolution photo viewer.

A starting point to a lot of information about similar RK devices can be found at [http://dtbnguyen.blogspot.com.au/2012/07/if-only-reading-were-easier.html](http://dtbnguyen.blogspot.com.au/2012/07/if-only-reading-were-easier.html)
