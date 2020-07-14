---
layout: post
slug: hacking-bluray-players-just-for-fun-1
title:  "Hacking BluRay players just for fun - part 1"
date:   2019-07-13 01:00:00
categories:
- infosec
tags:
- java
- bluray
- hacking
---


Most of my family keeps complaining whenever we watch a movie that the screen kept glitching. To solve this problem I bought a new BluRay player. They come with an Ethernet port these days so I decided to have a look around before deplying it to the loungeroom...

## Editorial?

Such viewing for many people these days is Fetch or NetFlix (which we have) or $STREAMING_SERVICE which is convenient but also it seems silly to have to pay for more than one subscription just to see different things and to not watch 95% of the stuff the do provide, but we are stuck with that for the moment. FWIW I recommend 'Messiah' on NetFlix as a good binge watch if you want something a bit kind of a cross between cold war spy thrillers and Forrest Gump. No, really, I actually did like it. So why I am I still replacing the BluRay? We have a collection of movies on disc which we like to rewatch, also our local general store still rents DVDs for $5 and I knod of like popping in for a chat and grabbing a movie to hire, its a bit old sckool but still fun!

Anyway I digress. Just for fun, I'm going to apply a some cybersecurity buzzwords I mean domain knowledge to practice a few other things I've ben studying in order to consolidate my own knowledge.

## Planning I mean Unboxing

So I bought a new, and really quite cheap (~ $130, or less with haggling) for what you get, not homebrand BluRay player from a certain aussie-wide vendor that I don't avoid, unlike hardly normal. The interesting thing is they provide firmware downloads on their webside and an update method over USB, which means I could download the firmware while the gadget was in the mail and unpack it with BinWalk just for a laugh. More on that later.

Specs:
- its actually smaller physically than it looks. Quite interesting really
- IR remote
- front USB port
- read Ethernet port
- 1x HDMI is the only output. I guess thats how these can be so much smaller, my previous player had Component and yellow/red/white A/V as well.
- As a bonus the power brick can be unplugged, which is a nice touch
- standby 0.5 watt - can't be avoided I guess
- the brick itself though is sideways which is really annoying, means you cant have another power plug next to it on one side without using a vertical double adaptor. sigh.
- the only buttons are Eject, and On/Off.
- allegedly it will play a laundry list of formats
- it would presumably be Region 4 - although there may be workarounds
- I didn' bother paying a couple $100 extra for the wireless version
- There is a 'Java Powered' logo on the back

TBH I wasn't planning to hook it up to the net anyway I already have an old laptop setup for NetFlix which comes in handy for other purposes. But I might consider it later.

So, the objective of this exercise is to see if there is anything interesting that might be done with this player besides watch a movie with the family. In terms of business objectives that outweighs everything else, so lets not break the thing...

## Discovery - Research

I downloaded the firmware for this.

What am I expecting to find?
- Hopefully a Linux operating system which means I could try and build and install additional toys - if I get time, which is not super likely sadly
- Java apps - as above

Lets take a look. But first, lets get hands on, because I feel like it!

## Discovery - Reconaissance

Rather than using it and then seeing what else, I've never even turned it on yet and I'm instead connecting it to my test laboratory. This is for fun after all.

Hypotheses:
- will it auto ask for DHCP on power up, before I've even turned it on connected to the Tele and configured it? Lets see
- will it have a static IP and if so can we work that out by what it spits out if any?

Note, I'm haven't read any manuals yet other than the quickstart guide for the specs, I probably should but for fun I'm going in blind.

```
 +-------------------+               +-----------------+
 |                   |               |                 |
 |                   |               |                 |
 |   Intel NUC   eth1+---------------+  BluRay player  |
 |                   |               |                 |
 |   Debian 10       |               |                 |
 |                   |               |                 |
 |       eth0        |               |                 |
 +-------------------+               +-----------------+
          |
          |
          v
       my network
```

(diagram created with [asciiflow](http://asciiflow.com/)

First task - log everything output on powerup.

I set up the NUC port (eth1 in the diagram, in practice something like enx00e04cabcdef as it is a USB adaptor) with a static IP, and have it alive even if the link is down.  This I have done previously using NetworkManager, as I used this for other testing, so I have multiple static IP addresses. To keep it simple also disable ipv6:

```
nmcli c add con-name "USB-ether-1" ifname enx00e04cabcdef type ethernet \
  ip4 10.5.5.1 ip4 192.168.0.253
nmcli c mod "USB-ether-1" ipv6.method ignore
nmcli c up "USB-ether-1"
```

Take care copying/pasting, I tried this from VSCode on Mac via ssh and it did strange things to the quotes and caused me all manner of pain for a while...

Now run tcpdump and lets see what we can get. This command saves to a pcap file and also dumps to the console:

```
tcpdump  -n -i enx00e04cabcdef -U -s 0 -w - | tee capture.pcap | tcpdump -r - -e -l -n
```

What this does:
- `-n` avoid DNS lookups
- `-U` send data through on each packet instead of until the buffer is full
- `-s 0` capture entire packets
- `-w -` send pcap to standard out
- `-e` dump link headers as well
- `-r -` read packets from stdin
- `-l` line buffer the output, usually needed when using tcpdump to avoid stuttering and make CTRL+C work as expected

All this lets us capture a pcap we could look at in Wireshark later, whilst also seeing the output.

So lets do it. Power on, and what happened?

- the NUC started outputting ipv6 router solicitation packets (annoying)
- then I saw data from the player - bootp requests (✅)
- after a minute, it gave up and switched to APIPA and output ARP packets to check 169.254.16.84 was OK to use
- then it started outputting IGMPv3 multicast records
- after a couple more minutes it again went into a bootp loop

Not completely unexpected.

In the meantime I modified my connection to see if I can pin the APIPA address

```
nmcli c mod "USB-ether-1" +ipv4.addresses "169.254.33.33/16"
nmcli d reapply enx00e04cabcdef
ping 169.254.16.84
```

That worked! Time to fire up nmap.

Note, all this time those bootp are still being recorded, as is the ping, so that will serve as a marker. The nmap scanning will also end up in the pcap file, so lots to sort through later.

Start by doing a quick TCP port scan.
```
nmap -p1-65535 -v -sT 169.254.16.84
```

Bingo:
```
Not shown: 65532 closed ports
PORT      STATE SERVICE
9222/tcp  open  teamcoherence
51623/tcp open  unknown
56789/tcp open  unknown
```

Lets try and guess the O/S:
```
nmap -A -p9222,51623,56789 -v -sT 169.254.16.84
```

This took its time. So whatever is listening is not handling the guess packets very well. A few minutes later:

```
PORT      STATE SERVICE VERSION
9222/tcp  open  http    Google Chromecast httpd
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Content shell remote debugging
51623/tcp open  http    Mongoose httpd
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-title: Site doesn't have a title (text/plain).
56789/tcp open  http    Mongoose httpd
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-title: Site doesn't have a title (text/plain).
MAC Address: B4:6C:47:08:D2:EC (Unknown)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Ruckus ZoneFlex R710 WAP (Linux 3.4) (98%), Linux 2.6.32 - 3.10 (98%), Linux 3.2 - 3.16 (96%), Linux 3.2 - 4.9 (96%), Linux 3.4 - 3.10 (96%), Synology DiskStation Manager 5.2-5644 (96%), Linux 2.6.32 - 3.5 (95%), Linux 2.6.32 - 3.13 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 2.6.32 (95%)
No exact OS matches for host (test conditions non-ideal).
```

Chromecast should be obvious I guest. Actually, quite handy - means I can possibly use this player to cast my iPad and Galaxy Tab screens, useful!

We can look at those web services later.

Linux makes sense. We can confirm that when unpacking the firmware upgrade file with luck.

At this stage I noticed the tcpdump stopped, something seems to have crashed after that scan... cant ping anymore either. Power cycle time.

After the reboot it got the same APIPA address, good to know.

Lets do a quick UDP scan. Not really quick, because it takes a few seconds for all 65535 ports to repond with unreachable (or timeout) - tcpdump is showung unreachables, so there is no firewall, at least, not one configured to drop. (Not that I expected a firewall on a BluRay player, but perhaps after this it might be something I can add. Not that I expect to have the time, but it might be feasible...)

```
nmap -p1-65535 -v -sU -Pn 169.254.16.84
```

Eventually, after 30 minutes -




## Discovery - Research (again)

## Security testing (safe for use)

Purpose of this is to check it doesnt phone home, or if it does what to block on my network. They don't need to know when I'm watching a DVD thats my business.

To start with, I'm putting an OpenWRT router in the way - that will add WiFi from my lounge room, and also let me block and log _everything_.

I'll analyse that in Part 2 of this block (TBA)