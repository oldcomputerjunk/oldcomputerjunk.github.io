---
layout: post
slug: cheap-smart-speaker-teardown-part1
title:  "Cheap Smart Speaker Teardown part 1"
date:   2022-11-12 01:00:00
categories:
- gps
tags:
- embedded
- hacking
- reversing
- android
- hardware
- home-automation
---

_Prelude: its been a while. Not that I've done nothing, but I kind of dropped the habit of writing publically about stuff here. Some stuff I wote up in standalone giuthub repos as well._

# Introduction

Last month I picked up a cheap smart speaker from a local electronics retailer and this weekend I was rained in, so took the opportunity to have some fun and practice.

# Scope

I have been trying not to buy so much junk the last while, but in this case the thing was on sale for $29 (apparently they retailed for $179 new a few years ago) and the long range weather forecast hinted at yet more rain and storms, so I figured the next weekend I was not abl to get out or into the garden I'd do another teardown, and this time, also, write it up... this weekend, it definitely rained. A lot. And the wind... so I had a good chunk of time. Cognisant of the rabbit hole though, so the scope here is limited to the physical tear down, black box reconaissance, and dumping the firmware and a preliminary attack surface analysis. There is opportunity to do some network analysis and also to reverse binaries, but that may be a project for the future. Or not, as there are plenty of activites to attract attention... and hopefully the weather will finally clear as we head into summer.

# Objective

Given the scope and timeframe, I want to gain access to a terminal (if possible) and dump the firmware, ideally without resorting to a soldering iron (if possible - at present my lab is in a state where I'm not currently equipped to desolder more than an 8-pin SOIC flash), and take a stab at where one might go next if sufficiently motivated in the future.

Secondary objective if at all possible, is disassemble it in a ay I can put it back together later :-)

# Preparation & Operating Theory

For my current timeframe, doing a network analysis is out of scope, so I dont want it connecting to the Internet or having to link an account (at least it appears to just need the google home app and no vendor app)  Thus, I'm working in the following order: tear down, power up, gain access, dump firmware.

I have my trusy Raspberry Pi 400 which I will use for a serial terminal (with luck), and a Wifi Access Point. According to the device instructions, I should be able to turn it on then find it with google home. I wont be doing that, so I'm going to try bluetooth scanning and wifi scanning - often IoT devices will expose an open Wifi AP for the initial pairing. I will make sure it doesn't connect to my normal AP, I dont want it accidentally upgrading the firmware.

# Part 1 - Physical Teardown

This looks like a not-completely-rubbish product on the outside, it would not look out of place in a high brand department store rather than the cheap stuff you find at Aldi or Kmart. It supports Home Assistant for issuing commands, and Chrome Cast, and can also operate as a plain old bluetooth speaker. Made in China of course.

This device feels like reasonable quality when you pick it up. It has a strap that feels not-fragile, and the specification claims to be IPX6 for splash resistance. It has a nice weight and looks like it might not completely break the first time you drop it on a concrete floor. There are four buttons for operation, and coloured LEDs. And one assumes, a speaker and microphone(s).

![Smart Speaker Top View](../images/cheap-smart-speaker-teardown-part1/10-smart-speaker.png){:height="50%" width="50%"}

The manufacturer has also learned a thing from Apple in how it was packaged.

![Packaging](../images/cheap-smart-speaker-teardown-part1/20-packaging.png){:height="50%" width="50%"}

On turning it over, there is one obvious screw cover, so I gently unstuck it. The tool shown is from a mobile phone repair kit.

![Strapping screw](../images/cheap-smart-speaker-teardown-part1/30-strap-screw.png){:height="50%" width="50%"}

However, that screw only seems to connect the strap, and also it is free-wheeling after I loosten it. Given the moisture-resistant nature of the device, I dont think that one actually opens it anyway. I confirmed my suspicions this was a red herring, when I started playing at the edge of the circular padding on the base of the device, which turned out to also be a fat sticker. I peeled that off and exposed four more screws. Also a hole with a clear sticker, which I'll get back to later.

![Sticker covering the screws](../images/cheap-smart-speaker-teardown-part1/40-screw-cover.png){:height="50%" width="50%"}

With the screws removed, we can see inside. The enclosure has two parts, with a PCB in the lower half with cabling connected to components in the upper half.

![Initial inside view](../images/cheap-smart-speaker-teardown-part1/50-inside.png){:height="50%" width="50%"}

There is a speaker, which actually looks (and later, felt) quite good quality. There is also a compartment for the battery, part of the IPX6 construction.

![Inside the top view](../images/cheap-smart-speaker-teardown-part1/55-inside-top.png){:height="50%" width="50%"}

I was then able to separate the enclosure halves. A wireless antenna (I'm assuming, Wifi) was attached with a sticky flex PCB antenna to the top, with a hot glue glob holding the U.FL connector to the PCB.

![The enclosure split into halves](../images/cheap-smart-speaker-teardown-part1/60-enclosure.png){:height="50%" width="50%"}

