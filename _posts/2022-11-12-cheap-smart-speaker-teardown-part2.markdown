---
layout: post
slug: cheap-smart-speaker-teardown-part2
title:  "Cheap Smart Speaker Teardown part 2"
date:   2022-11-12 02:00:00
categories:
- infosec
tags:
- embedded
- hacking
- reversing
- repurposing
- android
- hardware
- home-automation
---

# Introduction

Last month I picked up a cheap smart speaker from a local electronics retailer and this weekend I was rained in, so took the opportunity to have some fun and practice doing a teardown and firmware dump

This is the second part in a series:
- [Part 1]({{ site.baseurl }}{% link _posts/2022-11-12-cheap-smart-speaker-teardown-part1.markdown %}) - Photo essay of physical teardown

# Recap

To recap, the objective is to complete a physical tear down (done), perform a black box reconnaissance, dump the firmware and with luck, adapt the device to do some of the things a Raspberry Pi might do if the were not unobtanium at the moment! With luck, then put it back together retaining access to the running internals.

# Part 2 - Passive Reconnaissance

I mean, usually you might do this first, but really, I would usually do it in parallel because you find interesting hints while pulling things apart. In this case I stayed patient and captured a complete set of photos before I started.

The following is recorded in the order I was searching, not any particular logical order of attack...

## Chip identification

First because this is always interesting to me, I identified the integrated circuits. The image is the same one from Part 1.

![Antenna](/images/cheap-smart-speaker-teardown-part1/70-ics.png){:height="100%" width="100%"}

The table lists them all; I'll follow with some comments.

<div class="overflow-table"  markdown="1">

| Marking | Identified by me as | Data? | Confidence | Other References | Notes |
| ------- | ------------- | ----------              | ---------- | ----- | -- |
| (none) | Mediatek MT2601 ?? Dual core ARM 64-bit with RAM | Gongkai | 88% | | It is not wireless, so I'm assuming this is the main processor (ARM, maybe MIPS, but the 2601 is ARM), and possibly a memory chip(s)|
| Mediatek MT6630QN | Wireless SOC | Gongkai | 100% | <https://corp.mediatek.com/news-events/press-releases/mediatek-announces-mt6630-worlds-first-five-in-one-combo-wireless-connectivity-soc-for-mobile-devices> | Bluetooth 4.1, Wifi up to 802.11ac. Not the main processor (unlike some systems) |
| Mediatek MT6392A | Power Management IC (PMIC) | Gongkai | 100 % | <https://patchwork.kernel.org/project/linux-mediatek/patch/20190619142013.20913-8-fparent@baylibre.com/>| Has Linux device driver. Can even be purchased from ALI express|
| PCM5121 79TG4 A5XD | Audio driver (Stereo DAC) | <https://www.ti.com/product/PCM5121> | 100% |  | |
| MX30LF4G18AC-TI | 512Mx8 Flash memory (aka eMMC) | <https://www.farnell.com/datasheets/2815343.pdf> | 100% | | You can even buy these: <https://au.element14.com/macronix/mx30lf4g18ac-ti/flash-memory-4gbit-40-to-85deg/dp/3129220>|
| S1304 3112 | LED controller | https://datasheetspdf.com/pdf/1301432/SI-EN/SN3112-09/1| 75% | | |
| STM8S003F3 P6 7B22F HL 730 Y | 8-bit ARM micro, 16MHz with 8K flash | |100% | | Maybe used for controlling the pretty patterns the LEDs make via the 3112?  |
| 358 | LM358 2channel Op-Amp | | 75% | | It makes most sense; maybe it buffers input from the microphones |
| CS8509E 8H22 | Audio amp| | | | Can be found on eBay|

</div>

There is no datasheet and little information on the Mediatek chips. I was eventually able to infer quite a bit, I'll go through this process in a bit. I have never quite understood the purpose of this. 95% of integrated circuits have their datasheets published; after all, you want the engineers who influence buying decisions to know how to use them properly. Intel and AMD obviously live or die by Intellectual Property and yet you can get datasheets for i7's or Athlons or whatever.

The only thing that I could not work out or even guess was the S1304 3112; perhaps it is like the 358 and missing suffix markers I haven't thought of.

[Gongkai](https://www.bunniestudios.com/blog/?page_id=3107) is a term highlighted by open source hacker and maker Bunnie Huang, describing how a Chinese version of open source culture in relation to sharing technical information is different from both Western proprietary IP and open source. I've used it here, because I managed to stumble over enlightening information with some creative searching.

Whether the metal can hides an MT2601 or not, spoiler! In Part 3 after I found the serial port, I confirmed the system runs an ARM 64 bit dual core CPU and has access to 512MB DDR.

## FCC search

If you are following conventional wisdom you might do this before even starting a teardown. Nevertheless, it was there, as the product is sold in the states, and I was able to download an electronic version of the manual, although I already had it in hardcopy.

Notably, there is nothing in the manual (or the web) relating to the vendors GPL obligations. Spoiler - there is GPL code in the device (not least, in the kernel).

## Vendor search

They don't seem to be super community oriented citizens, aside from lack of GPL obligation fulfilment, there were lots of people asking for firmware updates on other products, with no obvious online place to download any updates from that I could see. (Spoiler: in practice it turns out the device self updates when you first pair with it)

Along the way I found a GitHub repo where someone had aggregated information on quite a few similar speaker products. This proved helpful.

## Preliminary search for code and CVE's

I did find some interesting things, I might come back to that later.

One thing that later turned out to be true, I guessed this device was running Linux, perhaps a flavour of Android - I cant see why it would need 512MB of flash otherwise.

## What about the main chips?

Coming back to Gongkai...

My main research tools starts with google like most, as the default in Safari in my iPad. I moved to DuckDuckGo pretty quickly go. In particular, often when searching from hints like chip numbers and the like, often google returns either nothing, or unrelated junk. I'm finding over recent years it getting worse and worse for many technical searches, not just this kind - the war of attrition between SEO and "the algorithm" has as a casualty the ability to simply search for a part number, or a Linux command, in isolation, for example, and not get 15 website clones of a blog that looks like it was generated using shitty machine learning from older (and now out of date) resources or stack overflow.

Anyway, rant aside, I found much more joy using the following strategy:
- using google image search to find hints, and even diagrams
- taking those hints and mixing with the original search in DuckDuckGo which seems to turn up different junk but also often stuff that google missed
- and occasionally even Yandex
- my hypothesis is that google indexing is tweaked for different parts of the globe; here, I'm looking for leakage and inferences from china, they are more likely to turn up in other places

Note, before going too far down dark holes, I switched over to ProtonVPN and a private safe browser... some stuff was already geo-blocked by my own firewall!

In the end though, I found a lot of information in a blog written in (probably) Chinese, with links to other stuff; and eventually found my way to a site with previews of a trove of PDFs. This site was also Chinese, and when I clicked on the image of what looked like the datasheet it wanted me to pay to downloaded it (AFAICT). However, there was a scrollable preview - so in the end, anyone with a snipping tool is now home! (a [Hint](https://www.docin.com/)!)

I also found interesting other things I found along the way, such as information related to GPIO programming, for example.

One thing that helps turn up other interesting things - some of the blogs had screenshots, with filenames - sometimes you can get luck searching for those filenames too... or other forms of google-dorking... that has been fruitful in the past, but I had less joy this time.

# Further research

At this point, I returned to the tools, broke out the DMM, oscilloscope and soldering iron, and hunted for the serial port.

---

I will come back and add in some extra links later.
