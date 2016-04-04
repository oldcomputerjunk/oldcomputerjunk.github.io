---
author: admin1
comments: true
date: 2011-12-17 03:59:37+00:00
layout: post
slug: optus-huawei-e173-or-e1752-in-debian-squeeze
title: Optus Huawei E173 in Debian Squeeze
wordpress_id: 41
categories:
- howto
- linux
tags:
- debian
- github
- huawei
- modem
- usb
---

Well as usual there was a storm and my fixed wireless "broadband" (pre-WiMax) stopped working, and of course it is a Saturday which often means no Internet until Monday.  Or at least it used to until we had various spare USB 3g dongles in the house.  So I finally bit the bullet and worked out how to use a space Optus Huawei E173 with my Debian Squeeze system... <!-- more -->

( In any case my wife dislikes the fixed wireless as it always seems unreliable for her, although this was in part to a dodgy Windows 7 driver for the Toshiba laptop built-in wifi which I have since fixed by upgrading the driver. She uses a 3g dongle 3/4 of the time - most recently I got a great deal from Optus with my last phone recontract where for $29 a month I not only had a new phone and capped plan but also with 1GByte/month capped E173 USB modem with free Facebook. )

When the fixed wireless stopped, seeing as I was trying to access work webmail I connected my work Telstra backup to my OpenWRT router and switched over, except unusually this time I was getting a lot of dropouts on Telstra as well. 
So instead I decided to get the Optus E173 running directly on my main box.  It should have plugged in and worked with NetworkManager, except it turns out that I used to use an E160E and had not in fact tried the E173, which didn't fire up as expected.

To cut to the chase it turns out that, like the Telstra Sierra USB modules, Linux detects the E173 not only as a bunch of serial ports but as an Ethernet port as well, `wwan0` in my case - this is accomplished using the `cdc_ether` kernel module, which auto loaded for me in squeeze.

So the solution:


  1. I have NetworkManager disabled.


  2. Edit `/etc/network/interfaces` and add the following:
[crayon lang="plain"]
allow-hotplug wwan0
iface wwan0 inet dhcp
[/crayon]



  3. Create a script, something like `~/bin/start-optus-e173.sh` (See Below)


  4. Execute script as sudo (See Below)


  5. I disconnect by unplugging, but you can issue an AT command to do that or write a script if you can be bothered.





### Update



I then hit the problem described [here](http://osdir.com/ml/comp.os.linux.hardware/2011-09/msg00015.html), where the wrong MAC address was set on the card!

The workaround is
[crayon lang="bash"]
ifconfig wwan0 hw ether 00:01:02:03:04:05
[/crayon] (script updated to do this automagically)



### Warning - this is probably insecure - you should set up iptables, and using the dodgy MAC could be dangerous. But this does for an emergency



**TL;DR**

Script for `~/bin/start-optus-e173.sh`:
[crayon lang="bash"]
#!/bin/bash
#
# Script to startup USB modem
# Could no doubt be rewritten to use EXPECT or something similar
# Also could do with a trap to cleanup for when its interrupted
# Signal strength is reported asynchronusly on ttyUSB2
#
# Ref: http://forums.freebsd.org/showthread.php?t=15952
# Ref: http://www.draisberghof.de/usb_modeswitch/bb/viewtopic.php?t=734
#
APN=PUT YOUR APN HERE
TTY=/dev/ttyUSB0
# set -x # Uncomment to debug scripts
set -e # Exit on fault in script

ifdown wwan0 && true > /dev/null 2>&1


# Report signal strength while working
# 0-31 where 31 is best but I get about 7 in the study
cat /dev/ttyUSB2 | grep RSSI &
PID=$!
cat /dev/ttyUSB0 | sed -e 's/^/LOG: /' &
PID2=$!

trap "kill $PID $PID2 ; exit 0" INT TERM EXIT

stty -F $TTY ispeed 406800 ospeed 406800 -echo

# Reset
echo -ne 'ATZ\r\n' > $TTY
sleep 1
echo -ne 'AT&F;\r\n' > $TTY
sleep 1
# Prepare to work
echo -ne 'AT+CFUN=1\r\n' > $TTY
sleep 1
echo -ne 'AT+CMEE=2\r\n' > $TTY
sleep 1
echo -ne 'AT+CSQ\r\n' > $TTY
sleep 1

# Print out available networks - 50502 is optus
# Should see: +COPS: 0,2,"50502",2
echo -ne 'AT+COPS?\r\n' > $TTY
sleep 1

# Enable wwan0 for DHCP - connects to the service
# At this point the blue led finally stops flashing and turns on solid
echo -ne 'AT^NDISDUP=1,1,"'$APN'"\r\n' > $TTY
sleep 1
ifconfig wwan0 hw ether 00:01:02:03:04:05
ifup wwan0
ifconfig | grep -A1 wwan0
kill $PID $PID2
exit 0
[/crayon]

For now it needs root privileges, I could probably add myself to the right group and make it work, but I cant be bothered. If you have selinux enable it would probably complicate things even further.

Successful Output:
[crayon lang="default" show-plain-default="true"]
^RSSI:4
^RSSI:4
^RSSI:4
LOG: 
LOG: OK
LOG: ATZ
LOG: OK
^RSSI:4
LOG: AT&F;
LOG: OK
LOG: AT+CFUN=1
LOG: OK
LOG: AT+CMEE=2
LOG: OK
^RSSI:4
LOG: AT+CSQ
LOG: +CSQ: 4,99
LOG: 
LOG: OK
LOG: AT+COPS?
LOG: +COPS: 0,2,"50502",2
LOG: 
LOG: OK
LOG: AT^NDISDUP=1,1,"CONNECTCAP"
LOG: OK
^RSSI:8
Internet Systems Consortium DHCP Client 4.1.1-P1
Copyright 2004-2010 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/wwan0/02:50:f3:00:00:00
Sending on   LPF/wwan0/02:50:f3:00:00:00
Sending on   Socket/fallback
DHCPDISCOVER on wwan0 to 255.255.255.255 port 67 interval 4
DHCPOFFER from 199.199.199.198
DHCPREQUEST on wwan0 to 255.255.255.255 port 67
DHCPACK from 199.199.199.198
Reloading /etc/samba/smb.conf: smbd only.
bound to 58.109.51.163 -- renewal in 2847 seconds.
wwan0     Link encap:Ethernet  HWaddr 02:50:f3:00:00:00  
          inet addr:199.199.199.198  Bcast:199.199.199.199  Mask:255.255.255.248

[/crayon]





