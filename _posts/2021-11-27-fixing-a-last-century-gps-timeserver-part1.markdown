---
layout: post
slug: fixing-a-last-century-gps-timeserver-part1
title:  "Fixing a last century GPS timeserver part 1"
date:   2021-11-20 01:00:00
categories:
- gps
tags:
- embedded
- hacking
- hardware
- gps
- retrocomputing
---

_(go another part 1!)_

This blog is a rough draft. I will write something up a bit neater with photos and PDFs of references on Hackaday.com, look for [@pastcompute](https://hackaday.io/pastcompute)

A number of years ago (maybe a decade ago) I acquired garage-sale style a 1RU GPS time server, this was from a company I'd worked for when they were wrapping up a project. It was an Odetics (now Zyfer?) GPStar 365, with a number of cool functions including a synchronized 10MHz frequency and 1-PPS output as well as normal NMEA-style text output on serial port.

Recently about 5 weeks ago I was having a clean up and came across it again, and decided what the hey lets put it to use as an accurate time source for my home lab. Of course, I powered it up and nothing... the week after I decided to make this a bit of a project, so I disassembled it and after some testing discovered that the power supply was flaky. I had it running when disassembled, then again, it was dead.

A week later, and having a look at the PSU discovered it was a fairly common form factor, it has +5 and +/-12V outputs and I could get one with the same connecter via Element 14 if i wanted to pay > $50 and wait a number of weeks. For this project not worth spending such sums, so I started looking at the power supply, tracing out some of the circuit in Kicad, and discovered it uses a common switch mode controller chip, a UC3842. Reading around on some forums, it was most likely a dead capacitor. During the week I acquired some from the nearest Altronics, along with a few other resistors and the like just in case, as there had been some heat wear evident on some components. One interesting thing I learned here is there are oddities in the circuit that make sense when you read the data sheet. For example, the chip needs above approx 11V to even start working, something to do with the inductance of the switch mode transformer, so it gets a DC-ish tap through a zener to bootstrap the chip before the main capacitor charges above 11V and things work properly. The other thing, which I did suspect and therefore treated the thing with safety precautions, and confirmed with a multimeter, was live rectified mains ~ 350-odd volts (Australian mains is 240AC) in the area.

Having replaced the capacitor and done some makeshift track repair work, fired it up, and whilst checking output voltaged managed to drop a tool on the board and BANG. Ok oops dead power supply, and if you do try this at home always take proper mains precautions - I use a rated multimeter, keep my hands away from things and work using ELB and a secondary protection board and even then be ready for things to go wrong, so dont put your body in a place to short. Of course, dropping a tool means it is not connected to me but it still managed to short and explode the bridge rectifier, the fuse, and a nearby NTC thermistor in the neutral line as well. Back to Jaycar for more parts. And Element14 because nobody keeps little 4-pin bridge rectifiers. The part I needed was also obsolete but I found one with a similar spec.

Another week later (I think four from the start) I sat down and soldered the bridge rectifier, being SMD this time I had to solder it on the back of the single sided PCB, also bend the pins up because it had to go on upside down, this time no mistakes and the system powered up.

After connecting antenna and waiting an hour, it fixated the satellites.

Except the time was in 1992 and the position about 0.2 degrees away from here...

After a bit more research it turns out this device is afflicted by something called the 1024-week bug - GPS wraps at this interval, and the date it shows was exactly 19 years, 6-odd months and a bit incorrect, so just joy, not, and more work to do.

My options:
- as pointed out to me by a friend, it was 'working' with a nice 1-PPS and sychronized 1MHz clock, so I could just hook the 1-PPS and the serial port to a Raspberry Pi and correct the time on the way into NTP.
- or I could go down a rabbit hole and try and fix the firmware!

Some version of #1 will be needed as this only has BNC and RS232 outputs, to get the time into NTP.
But I want the right time on the display too! So lets see how the rabbit hole goes.

I had had a look at the main PCB when the lid was off and discovered the following;
- there is a Magellan GPS module separate to main PCB
- main PCB has an 80C188 (an almost-microcontroller version of the 8088/8086), and 88c681 dual UART, a 93C46 256 bit EEPROM and a bunch of FPGAs
- also some 74series 245 / similar chips probably for driving the LCD and feed I/O to other components including LEDs, dip switches and the like, and the bus to the magellan and the FPGS
- at a guess I'd say the magellan itself outputs the 1-PPS and 1MHz clock and everything else works off that
(Spoiler - after I'd reverse engineered the firmware, it turned out there was a 1kHz clock connected to an external interrupt line driving the time displayed on the LCD and the serial port... likely derived from the 1MHz signal)

The 80C188 is half way between a full CPU, needing a full suite of support chips, and a modern microcontroller where everything is on board. Probably more like a SOC but without the GPIO. So this simplified embedded design in the 90's all you need is RAM (there is a 128KB static ram) ROM (there is 128kB EPROM, a 27c010, thankfully) and there are timers and an interrupt controller on the chip (the 8086 they were separate) and also logic for managing chip selects, and also the data bus is only 8 bits which simplifies the peripherals. It also adds a few extra instructions to the x86 instruction set over the plain 8086.
 
In part #2 I dump the EPROM - using a raspberry pi because I discovered I no longer had a usable EPROM programmer - and reverse engineered the firmware, making a number of interesting discoveries. Hopefully I can then patch it to display the proper time, and perhaps there may be a #3 where I make a hardware mod to put the serial control inside the box and mount a Raspberry Pi inside...

References:
- 80C188 Embedded Microprocessor 8086 compatible
- 88C681 Dual UART
- UC3842 Switch Mode Powersupply Controller
- GPStar 365 manual
- SN1B60 Bridge rectifier chip (SN) and what I replaced it with (DFL1506S-E3/77)


