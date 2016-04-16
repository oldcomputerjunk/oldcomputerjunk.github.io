---
author: admin1
comments: false
date: 2014-02-23 12:43:09+00:00
layout: post
slug: using-a-brother-network-scanner-with-linux
title: Using a Brother network scanner with Linux
wordpress_id: 491
categories:
- howto
- linux
tags:
- linux
- scanner
---

For a while now we have had a Brother MFC-J415W all in one wireless printer / fax / scanner thingy. It prints fine using CUPS and we have used it as a scanner with SD cards in sneaker-net mode.



## Linux support



Brother actually do a reasonably good job supporting all of their recent equipment under Linux.  All the drivers for CUPS and for the scanner functionality are available as packages for Debian, with usable documentation, from the Brother website [1].

I finally got around to setting things up for network scanning.

First you need to install the scanner driver; in my case it was the brsaneconfig3 driver.  You then set this up as follows:
```bash
sudo dpkg -i brscan3-0.2.11-5.amd64.deb
sudo brsaneconfig3 -a name="MFCJ415W" model=MFC-J415W \
                    nodename=printerhostname
```

Running `brsaneconfig -q` will output a (large number of) supported devices, and any it finds on your network:
```bash
$ brsaneconfig3 -q


Devices on network
  0 MFCJ415W            "MFC-J415W"         N:printerhostname
```

You can then run up gimp or your favourite graphical / scanner tool and test that scanning works as expected.

Having done this, I then set up remote scanning.  This involved running a service on the destination computer, which I set up to run as the logged in user from the openbox autostart. For some reason the tool (undocumented) requires the argument '2' to get help to show... The simplest method is as follows:

```bash
$ sudo dpkg -i brscan-skey-0.2.4-1.amd64.deb
$ brscan-skey -h -2
$ brscan-skey -l
 MFCJ415W          : brother3:net1;dev0  : 192.168.1.14         Active
$ brscan-skey -a MFCJ415W
```

After this setup, you simply need to run `brscan-skey` when you want to scan to your PC.  From the MFC LCD panel, you choose 'Scan', 'File' and then it should find your computer, and it displays the current user name as the computer for some reason.

Files get saved into $HOME/brscan by default.



## Improved remote scanning integration



Well of course I didn't want to stop there.  To make the system more user friendly, we expect:
* notifications when a document is received
* conversion to a useful format - the default is PNM for some reason, possibly this is the native scanner output

So I wrote a script, which starts with my X/session for openbox.

This is the essentials:
```#!/bin/bash
#
# Needs: apt-get install imagemagick 

PDF_DEST="$HOME/Scanned PDF"
LOG="$HOME/brscan/brscan.log"

brscan-skey | while read -r msg ; do 
  
  F="`sed -e 's/^\(.*\) is created\..*$/\1/' <<< $msg`"
  FB="${F%%.pnm}"
  B=`basename "$F"`
  BB=`basename "$FB"`
  D=`dirname "$F"`

  date >> $LOG
  echo "F=$F" >> $LOG

  notify-send -t 5000 "Brother MFC-J415W" "Received: brscan/$B"

  Y="Failed: MISSING INPUT FILE"
  test -f "$F" && Y=`convert "$F" "$FB.jpg" 2>&1`
  notify-send -t 5000 "Brother MFC-J415W" "Conversion to $FB.jpg\n${Y:-OK}"
  echo "$FB.jpg Y=$Y" >> $LOG

  Y="Failed: MISSING INPUT FILE"
  test -f "$F" && Y=`convert -page A4 -density 100 "$F" "$PDF_DEST/$BB.pdf" 2>&1`
  notify-send -t 5000 "Brother MFC-J415W" "Conversion to $BB.pdf\n${Y:-OK}"
  echo "$PDF_DEST/$BB.pdf Y=$Y" >> $LOG
done

notify-send -t 5000 "Brother MFC-J415W" "brscan-skey died for some reason…"
```

The way this works is as follows:
* brscan-skey outputs received file information on stdout
* so we run it in a forever loop and parse that information
* The `notify-send` program will popup a notification in your desktop window manager
* I then convert from PNM to both JPEG and PDF using imagemagick
* I also keep a log

I have pushed this script to the blog github repository, [https://github.com/oldcomputerjunk/blogscripts](https://github.com/oldcomputerjunk/blogscripts)

[1] [http://welcome.solutions.brother.com/bsc/public_s/id/linux/en/index.html](http://welcome.solutions.brother.com/bsc/public_s/id/linux/en/index.html)
