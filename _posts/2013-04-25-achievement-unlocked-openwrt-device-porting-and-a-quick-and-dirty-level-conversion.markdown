---
author: admin1
comments: false
date: 2013-04-25 14:32:30+00:00
layout: post
slug: achievement-unlocked-openwrt-device-porting-and-a-quick-and-dirty-level-conversion
title: Achievement unlocked - OpenWRT device porting - and a quick and dirty level
  conversion
wordpress_id: 418
categories:
- howto
- tech
tags:
- electronics
- hacking
- hardware
- openwrt
---

## Porting a Linux Router


So I just posted what I hope this time are properly formatted patches to the [OpenWRT](https://openwrt.org/) developers mailing list, a set of patches that add support to OpenWRT for the D-link DIR-632-A1 router.

You can see some of my progression on the OpenWRT [forum](https://forum.openwrt.org/viewtopic.php?pid=199406).

This was quite a bit of a journey, as I had a few false starts and along the way crossed a number of different areas in the Linux kernel. The best thing was debugging bit-banging of device registers, which reminds me of hacking on my Commodore 64 many years ago. I have of course had occasion to do such at times professionally, but I have rediscovered my desire to do more low level hacking, and having recently received a Raspberry Pi as a gift and finding use for my leostick from the 2012 linux conf I expect to have plenty of opportunity when I manage to find some more spare time.

Along the way I extended significantly my knowledge of how to work with OpenWRT and build customised images, etc. I added a [build guide](http://wiki.openwrt.org/doc/howtobuild/dir-632-a1.build) for this router to the wiki pending acceptance of patches into the main stream; I think this cuts through the useful but verbose documentation elsewhere in the wiki.


## When you need a USB-TTL RS232...


and you live 45m from Jaycar, and its 9pm Friday night anyway and you certainly don't want to wit 3days for Internet delivery from your friendly Aussie electronics suppliers (hello Core electronics!) or a month for some Chinese no-brand on ebay... and you want to hack now!

A crucial part of hacking a router is access to the serial port.  These are always at logic level, and in the case of the DIR-632 at 3v3 not 5V, further complicating things.  However all I had handy was a USB-RS232 dongle running at proper RS-232 levels, of +/- 12V - so connecting this would be sure to let the smoke out of something...

Now it turns out I have a shedful of what most people would term junk.  But usefully this included a tube of MAX232 chips I acquired some years ago.  So first, find a handful of resistors and capacitors, and a birdnest later, one logic level to RS232 converter:

[![nest](http://blog.oldcomputerjunk.net/wp-content/uploads/2013/04/nest-300x225.jpg)](http://blog.oldcomputerjunk.net/wp-content/uploads/2013/04/nest.jpg)

There is a leostick there, but it is being used to provide 5V :-) Originally I attempted to use it as a serial relay, but apparently there is a bug in the firmware in the model I have that causes problems with the d0/d1 serial port and thus all I could ever see was junk... (thanks to MarkJ for letting me know)

The circuit is basically as described [here (1)](http://www.scienceprog.com/alternatives-of-max232-in-low-budget-projects/), but I used 2x 2u2 capactitors in parallel, not having enough 1uF (* I really get into going the whole bush-mechanic-electronics thing!)

However, this doesnt fix the whole problem - the MAX232 is powered by 5V, although I also later found out it may have worked at 3v3 anyway.

I suspect many of the younger maker crowd might not be aware of some trick that can be employed when necessary: you can of course buy a 2cm sized 3v3 to 5v level converter form Jaycar, or from China on ebay, but again I didnt want to wait, and I had a pile of BC547 transistors handy and basic knoweldge of how logic circuits are actually built.

Thus, two transistors and a few resistors later, a logic level converter:

    
                  -+-----------------+- 5V
                   |                 |
                   |                 Z
                   |                 Z
                   |                 Z
                   |                 Z        __
                   |                 |     __|  |__
                   Z                 |
                   Z                 +--------->
         __        Z   __    __      |
      __|  |__     Z     |__|      | /
                   |               |/
                   +----/\/\/\-----|
                   |               |\
                 | /               | \
         4k7     |/                  V
    ---/\/\/\----|                   |
                 |\                  |
                 | \                 |
                   v                 |
                   |                 |
                   +-----------------+
                   |
                   |
                 --+--
                  ---
                   -


(ASCII art FTW!)

Going he other way, I just ran three signal 1N4148 diodes in series, this dropped the line to about 3v5 which managed to not blow anything up with!



And several rather late nights^H^H^H early mornings later:

[crayon lang="bash"]andrew@atlantis3:~/develop/openwrt/repos/openwrt$ telnet 192.168.1.1
Trying 192.168.1.1...
Connected to 192.168.1.1.
Escape character is '^]'.
=== IMPORTANT ============================
Use 'passwd' to set your login password
this will disable telnet and enable SSH
------------------------------------------

BusyBox v1.19.4 (2013-04-25 20:02:39 CST) built-in shell (ash)
Enter 'help' for a list of built-in commands.

_______ ________ __
| |.-----.-----.-----.| | | |.----.| |_
| - || _ | -__| || | | || _|| _|
|_______|| __|_____|__|__||________||__| |____|
|__| W I R E L E S S F R E E D O M
-----------------------------------------------------
BARRIER BREAKER (Bleeding Edge, r36419)
-----------------------------------------------------
* 1/2 oz Galliano Pour all ingredients into
* 4 oz cold Coffee an irish coffee mug filled
* 1 1/2 oz Dark Rum with crushed ice. Stir.
* 2 tsp. Creme de Cacao
-----------------------------------------------------
root@OpenWrt:/#
[/crayon]
--

Links:

(1) [http://www.scienceprog.com/alternatives-of-max232-in-low-budget-projects](http://www.scienceprog.com/alternatives-of-max232-in-low-budget-projects)
