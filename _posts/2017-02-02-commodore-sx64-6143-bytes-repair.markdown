---
layout: post
slug: commodore-sx64-6143-bytes-repair
title:  "Commodore SX64 memory addressing logic repair"
date:   2017-02-02 01:00:00
categories:
- tech
tags:
- retrocomputing
- commodore-64
---

# My Commodore SX64 has only 6143 bytes free (continued)

<img src="/public/sx64-k1.jpg" alt="6143 bytes free screenshot" class="inline"/>

[Previously](/2017/commodore-sx64-only-6143-byes-free) I blogged about cleaning the keyboard of my Commodore SX64, and discovering that it has some kind of memory access fault.

Before starting disassembly I decided to analyse the [circuit](ftp://ftp.zimmers.net/pub/cbm/schematics/computers/c64/sx-64/SX-64_Portable_Computer_Schematics.pdf) and try and narrow down the fault.

1. Starting hypothesis: the RAM per-se is hopefully OK but the addressing is faulty.

2. The C64 RAM is comprised of 8, 64kbit (8kByte) DRAM chips, the 4164. These only have 8 physical address lines, multiplexed into columns and rows to address all 65536 bits individually. There are two 4164 address strobe lines, /CAS (Column Address Strobe) and /RAS, for this purpose. We know that the C64 [kernal](https://en.wikipedia.org/wiki/KERNAL) will have determined that there are 6143 free basic bytes through having succesfully tested all available RAM up to that point, byte 8191, and not beyond. Thus the lower 8-bits of the address space of the 4164's (0-255 bytes) are OK, as are the CPU bus A0-A7. (Note: I need to look up whether /CAS or /RAS is driving the lower 8, but at the moment it doesn't matter.)

3. The range A0..A12 addresses the first 8192 bytes. Given the first 8192 bytes tested OK, this implies that the next 5 address bus bits A8-A12, along with the lower 5 multiplexed address bits used to access the RAM chips are operative.

4. Multiplexing of the lower and upper 8 address bits is managed by a pair of 74LS257 4-bit mux chips. When /SELA is low, pair B is loaded onto the mux output and thus the 4164 address bus, corresponding to A0-A7, and at other times, A8-15 will be present on the bus. In either case, the ouptut is tri-state when AEC is low and valid when AEC is high (the 74LS257 /OE is an inverting input and AEC passes through a 7406 inverter)

5. The upper address lines A13-A15 from the 6510 CPU itself are probably fine or the computer would not boot at all, because the BASIC and KERNAL ROM exist in ranges $a000..$c000 and $e000..$f000 and the I/O ports are at $d000.

6. AEC is probably working or the system would be dead as all RAM would be inaccessible

7. The input to the 4164 /CAS and /RAS are probably working or the system would be dead as all RAM would be inaccessible

A confusing thing to note: /CAS on the 4164 is driven by /CASRAM output from the PAL, whereas there is a second line /CAS on the circuit board which drives /SELA on the 74LS257's. These are connected in with the 6567 VIC (video chip) as there is some complexity around dynamic RAM refresh which is **TL;DR** and I hope not related to the problem.


Hypothesis: given the above, it is possible that MA5..MA7 at the connection to one or more of the RAM chips, are held low, hence only 8kb of RAM functional - but this can only be the case when /SELA is active, otherwise the computer would fail to boot at all.


Hypothesis: the RAM circuitry is fine, but the addressing logic is causing one of the ROM to connect to the bus when RAM should be accessed, and this is corrupting RAM data.

![Schematic around 74LS257 UA3](/public/sx64-s1.png)

It might be feasible to confirm this fault using a memory test program, as demonstrated in one of the forum discussions I linked previously. So I'll try that first. Physically, along with buzzing out all the lines in this area, I'll try using a logic analyser to confirm if this is the case.
