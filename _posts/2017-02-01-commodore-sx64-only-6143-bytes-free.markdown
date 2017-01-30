---
layout: post
slug: commodore-sx64-only-6143-byes-free
title:  "My Commodore SX64 has only 6143 bytes free"
date:   2017-02-01 01:00:00
categories:
- tech
tags:
- retrocomputing
- commodore-64
---

# My Commodore SX64 has only 6143 bytes free

<img src="/public/sx64-k1.jpg" alt="6143 bytes free screenshot" class="inline"/>

Quite a number of years ago I managed to aquire a [Commodore SX64](https://en.wikipedia.org/wiki/Commodore_SX-64) for the round sum of $10.
For the uninitiated a Commodore SX64 is a portable version of the hugely popular [Commodore 64](https://en.wikipedia.org/wiki/Commodore_64). It features a built-in version of the 1541 5.25" floppy disk drive, a 5" full colour CRT screen and all the normal Commodore 64 peripheral ports, with the exception of lacking analogue inputs on the joystick ports. It weights about 15 KG.

Finding one for $10 was quite lucky, even in the early 2000s when I stumbled across this at a garage sale. These computers were manufactured from 1984 to 1986 and is quite novel, as a luggable system, and reasonably rare compared with the C64 and Vic20, and although I have no record of it I suspect even then were selling for three figures on ebay. More recently (in about 2013-ish I think) one sold for around $400 roughly in my area. TBH I dont think there would be too many more around here.

<img src="/public/sx64-k0.jpg" alt="Commodore SX64" class="inline"/>

Mine is in quite good physical condition, having been stored in plastic in a dry relatively thermally stable
environment ever since. I have recently had it out to assess its condition, as I am pondering finding a new home for it.


The first thing I discovered after powering on was that the keyboard was mostly not working. The keyboard is essentially
the same layout as a normal C64, but is
designed to fit snugly in the front area of the case, acting as protection for the screen when in transit. The
keyboard is connected via a ubiquitious D25 connector, the keyboard cable having a presumably custom shell. (Whilst researching I found plans on [Thingiverse](http://www.thingiverse.com/thing:1561848) so presumably these break for some people.)

Luckily there were a [few](http://www.azog.org/?p=862) [tutorials](https://imgur.com/a/VP0Mm) around on how to deal with the failing keyboard, it seems
the most likely culprint was simply age and could be fixed with a good clean. I spent a couple of hours
disassembling the keyboard and applying isopropyl alcohol with some cotton buds, and this was sufficient to
return the keyboard to full functionality. The construction is of a spring activated key buttons and a carbon-paper like membrane, which I took extra care when dealing with as it would likely be impossible to find a
replacement.

<img src="/public/sx64-k2.jpg" alt="Commodore SX64" class="inline"/>

<img src="/public/sx64-k3.jpg" alt="Commodore SX64" class="inline"/>

<img src="/public/sx64-k4.jpg" alt="Commodore SX64" class="inline"/>


Actually it ended up taking about 3 weeks, because in spite of the warning in at least one tutorial I managed to
send a spring from one key into orbit somewhere in my workshop. Although I could find no supplier of
"Commodore SX64 keyboard spring" this did not matter and only cost me time, I was able to purcahse for a few dollars
springs of 1.8mm diameter from one of many Chinese ebay vendors and simply cut it to length.

After all this I was able to start the search for a floppy disk to test the system.

- First problem: booting with any cardridge would fail with a blank screen.

- Second problem: of several disk I tried, only one managed to play a game.

Now of course,
the odds of at least one or most of the 30+ year old floppies failing were quite high, but it
many would actually start to load, and then either hang in the game fast loader, or run with
corrupted graphics. It was during this testing that I noticed that there was only 6143 bytes free and not the expected 38911 :fearful:

THe go-to fault of course is a problem with the memory chips. This would be annoying, because although straightforward enough, at this point I really dont feel like desoldering, socketing and testing
8 4164 RAM chips, and finding a replacement for any that have failed.

Luckily it turns out that 6143 bytes corresponds to exactly 8KB of functional memory, and this [(link)](http://www.lemon64.com/forum/viewtopic.php?p=668304)   [(link)](http://www.lemon64.com/forum/viewtopic.php?p=502109) may actually be an addressing issue and not a failed RAM chip - in the best case a problem with either the address circuit lines or 74-series switching logic (good, because parts easily sourced) or if I'm  _unlucky_ it will be the addressing PLA :tired_face:

_to be continued_
