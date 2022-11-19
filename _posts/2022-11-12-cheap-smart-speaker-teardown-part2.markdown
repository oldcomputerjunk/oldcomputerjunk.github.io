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
- android
- hardware
- home-automation
---

# Introduction

Last month I picked up a cheap smart speaker from a local electronics retailer and this weekend I was rained in, so took the opportunity to have some fun and practice doing a teardown and firmware dump

This is the second part in a multi-part series:
- [Part 1]({{ site.baseurl }}{% link _posts/2022-11-12-cheap-smart-speaker-teardown-part1.markdown %}) - Photo essay of physical teardown
- Part 2 - passive reconaissance and integrated circuit identification
- [Part 3]({{ site.baseurl }}{% link _posts/2022-11-12-cheap-smart-speaker-teardown-part2.markdown %}) - find the serial port
- Part 4 - initial exploration (coming)
- Part 5 - (coming)

# Recap

To recap, the objective is to complete a physical tear down (done), perform a black box reconaissance, dump the firmware and make a preliminary attack surface analysis. With luck, then put it back together retaining access to the running internals.

# Part 2 - Passive Reconaissance

I mean, usually you might do this first, but really, I would usually do it in parallel because you find interesting hints while pulling things apart. In this case I stayed patient and captured a complete set of photos before I started.

The following is recorded in the order I was searching, not any particular logical order of attack...

## Chip identification

First because this is always interesting to me, I identified the integrated circuits. The image is the same one from Part 1.

![Antenna](/images/cheap-smart-speaker-teardown-part1/70-ics.png){:height="100%" width="100%"}

The table lists them all; I'll follow with some comments.

<div class="overflow-table"  markdown="1">

| Marking | Identified by me as | Data? | Confidence | Other References | Notes |
| ------- | ------------- | ----------              | ---------- | ----- | -- |
| (none) | Mediatek MT2601 Dual core ARM 64-bit with RAM | Gongkai | 88% | | It is not wireless, so I'm assuming this is the main processor (ARM, maybe MIPS, but the 2601 is ARM), and possibly a memory chip(s)|
| Mediatek MT6630QN | Wireless SOC | Gongkai | 100% | <https://corp.mediatek.com/news-events/press-releases/mediatek-announces-mt6630-worlds-first-five-in-one-combo-wireless-connectivity-soc-for-mobile-devices> | Bluetooth 4.1, Wifi up to 802.11ac. Not the main processor (unlike some systems) |
| Mediatek MT6392A | Power Management IC (PMIC) | Gongkai | 100 % | <https://patchwork.kernel.org/project/linux-mediatek/patch/20190619142013.20913-8-fparent@baylibre.com/>| Has Linux device driver. Can even be purchased from ALI express|
| PCM5121 79TG4 A5XD | Audio driver (Stereo DAC) | <https://www.ti.com/product/PCM5121> | 100% |  | |
| MX30LF4G18AC-TI | 512Mx8 Flash memory (aka eMMC) | <https://www.farnell.com/datasheets/2815343.pdf> | 100% | | You can even buy these: <https://au.element14.com/macronix/mx30lf4g18ac-ti/flash-memory-4gbit-40-to-85deg/dp/3129220>|
| S1304 3112 | ? | | 0% | | |
| STM8S003F3 P6 7B22F HL 730 Y | 8-bit ARM micro, 16MHz with 8K flash | |100% | | Maybe used for controlling the pretty patterns the LEDs make? |
| 358 | LM358 2channel Op-Amp | | 75% | | It makes most sense; maybe it buffers input from the microphones |
| CS8509E 8H22 | Audio amp| | | | Can be found on eBay|

</div>

There is no datasheet and little information on the Mediatek chips. I was eventually able to infer quite a bit, I'll go through this process in a bit. I have never quite understood the purpose of this. 95% of integrated circuits have their datasheets published; after all, you want the engineers who influence buying decisions to know how to use them properly. Intel and AMD obviously live or die by Intellectual Property and yet you can get datasheets for i7's or Athlons or whatever.

The only thing that I could not work out or even guess was the S1304 3112; perhaps it is like the 358 and missing suffix markers I haven't thought of.

[Gongkai](https://www.bunniestudios.com/blog/?page_id=3107) is a term highlighted by open source hacker and maker Bunnie Huang, describing how a Chinese version of open source culture in relation to sharing technical information is different from both Western proprietary IP and open source. I've used it here, because I managed to stumble over enlightening information with some creative searching.

Whether the metal can hides an MT2601 or not, spoiler! In Part 3 after I found the serial port, I confirmed the system runs an ARM 64 bit dual core CPU and has access to 512MB DDR.

## FCC search

If you are following conventional wisdom you might do this before even starting a teardown. Nevertheless, it was there, as the product is sold in the states, and I was able to download an electronic version of the manual, although I already had it in hardcopy:
- <https://fccid.io/2AHEA-TICHOMEMINI/User-Manual/User-Manual-3620025>

Notably, there is nothing in the manual (or the web) relating to the vendors GPL obligations. Spoiler - there is GPL code in the device (not least, in the kernel). Maybe we will come back to this in a later part.

## Vendor search

They don't seem to be super community oriented citizens, see GPL comment above, typical damnable IoT startup. Lots of people asking for firmware updates on other products, no obvious online place to download any from that I could see:
- <https://forum.mobvoi.com/viewtopic.php?f=2&t=2526&sid=6707ef83fa59b7f7321364c0875d7ae9&start=20>

Along the way I found a github repo wheresome had aggregated information on quite a few similar speaker products.
This increased the confidence I had that the CPU is an MT2601:
- <https://github.com/crifan/smart_speaker_disassemble_summary/blob/5c226921cd1538a0a5a28e452ef376536d4e45ee/src/common_smart_speaker/cpu_soc.md>

## Preliminary search for code and CVE's

A nice objective would be to find a pre-existing way to get into the system without needing to solder serial cables.
I didn't achieve that, because once I found the Uart pins I just wanted to get in and see what the system was (Part 3). But I did find the following CVE's, maybe they might be of use in the future:
- The Bluetooth component - <https://cve.report/software/mediatek/mt6630>

There are various MTK wireless drivers in the Android source code: I found much better examples later, but this turned up early:
- <https://android.googlesource.com/kernel/mediatek/+/android-mtk-3.18/drivers/misc/mediatek/connectivity/Kconfig>

And as noted above, the PMIC has drivers in Linux:
- <https://patchwork.kernel.org/project/linux-mediatek/patch/20190619142013.20913-8-fparent@baylibre.com/>

A CVE references the MT2601 in relation to Android 11 - from this and other inferences I wonder if this device is running some kind of Android derived O/S? (spoiler - it's derived from Chromium OS - cast devices actually run a headless web browser for fetching media - and need to have "permission" (and software) from google to run cast servers, as opposed to clients)
- <https://nvd.nist.gov/vuln/detail/CVE-2022-21775/change-record?changeRecordedOn=07/13/2022T12:37:40.630-0400>

## What about the main chips?

Coming back to Gongkai...

My main research tools starts with google like most, as the default in Safari in my iPad. I moved to DuckDuckGo pretty quickly go. In particular, often when searching from hints like chip numbers and the like, often google returns either nothing, or unrelated junk. I'm finding over recent years it getting worse and worse for many technical searches, not just this kind - the war of attrition between SEO and "the algorithm" has as a casualty the ability to simply search for a part number, or a linux command, in isolation, for example, and not get 15 website clones of a blog that looks like it was generated using shitty machine learning from older (and now out of date) resources or stack overflow.

Anyway, rant aside, I found much more joy using the following strategy:
- using google image search to find hints, and even diagrams
- taking those hints and mixing with the original search in DuckDuckGo which seems to turn up different junk but also often stuff that google missed
- and occasionally even yandex
- my hypothesis is that google indexing is tweaked for different parts of the globe; here, I'm looking for leakage and inferences from china, they are more likely to turn up in other places

Note, before going too far down dark holes, I switched over to ProtonVPN and a private safe browser... some stuff was already geoblocked by my own firewall!

In the end though, I found a lot of information in a blog written in (probably) Chinese, with links to other stuff; and eventually found my way to a site with previews of a trove of PDFs. This site was also Chinese, and when I clicked on the image of what looked like the datasheet it wanted me to pay to downloaded it (AFAICT). However, there was a scrollable preview - so in the end, anyone with a snipping tool is now home! (a [Hint](https://www.docin.com/)!)

Some interesting other things I found along the way:
- <https://www.cxymm.net/article/szx940213/83621319> -- GPIO information
- <https://www.cxymm.net/article/szx940213/85780909>
- <https://forum.xda-developers.com/t/mediatek-wifi-bt-fm-gps-combo-chips-hidden-capabilities-mt6620-mt6628-mt6630.3591182/>
- <https://www.programmersought.net/article/332836317.html?ysclid=ladxxdhdje376871861> -- only because of yandex
- <http://bbs.16rd.com/citiao.html>

Github is also a friend for potential treasure troves:
- <https://github.com/bkerler/mtkclient>
- <https://github.com/bq/aquaris-E5>
- <https://github.com/aquette/SP-Flash-Tool-source>
- <https://github.com/manviirr/Customization_Kit_buildspec>

One thing that helps turn up other interesting things - some of the blogs had screenshots, with filenames - sometimes you can get luck searching for those filenames too... or other forms of google dorking... that has been fruitful in the past, but I had less joy this time.

# Further research

At this point, I returned to the tools, broke out the DMM, oscilloscope and soldering iron, and hutned for the serial port. This is up next, in [Part 3]({{ site.baseurl }}{% link _posts/2022-11-12-cheap-smart-speaker-teardown-part3.markdown %})
