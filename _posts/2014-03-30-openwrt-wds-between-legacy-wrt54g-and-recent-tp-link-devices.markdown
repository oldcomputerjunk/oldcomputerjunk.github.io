---
author: admin1
comments: false
date: 2014-03-30 10:32:41+00:00
layout: post
slug: openwrt-wds-between-legacy-wrt54g-and-recent-tp-link-devices
title: OpenWRT WDS between legacy WRT54G and recent TP-Link devices
wordpress_id: 501
categories:
- howto
tags:
- network
- openwrt
---

For a while now I had a multiple wifi routers all providing access points, and a connection to each other, using a feature called [WDS](http://en.wikipedia.org/wiki/Wireless_distribution_system). All of the routers run OpenWRT. Recently one of them died and everything kind of stopped working properly. I actually had the following configuration:

`TP-LINK <--wired,bridged--> ASUS WL500G <--wireless,WDS,bridged--> Linksys WRT54G`

Now this all worked because the conventional wisdom appears to be that WDS works when the device at each end is the same, i.e. Broadcom to Broadcom (which I had in the case of WL500G and WRT54G) and is problematic at best otherwise.

Documentation alluding to this includes:

[http://wiki.openwrt.org/doc/recipes/broadcomwds](http://wiki.openwrt.org/doc/recipes/broadcomwds)

[http://wiki.openwrt.org/doc/recipes/atheroswds]( http://wiki.openwrt.org/doc/recipes/atheroswds)

[http://wiki.openwrt.org/doc/howto/clientmode]( http://wiki.openwrt.org/doc/howto/clientmode)

So with the WL500G out of the picture, the TP-Link and WRT54G were no longer able to see each other, and I experienced a loss of connectivity from my workshop where the WRT54G was physically located.

However if you look under the covers a bit, it seems this configuration can be made to work, [as suggested by the folk](http://www.dd-wrt.com/wiki/index.php/WDS#Broadcom_to_Atheros_WDS_Configuration_.28Unsupported.29) over at DD-WRT.

So with a bit of tweaking I was able to get WDS working between the TP-LINK and WRT54G as follows.


### TP-LINK Router (WR1043ND, OpenWRT 10.03 Backfire)


Content of /etc/config/wireless:

```plain
config 'wifi-device' 'radio0'
	option 'type' 'mac80211'
	option 'macaddr' 'xx:xx:xx:xx:xx:xx'
	option 'channel' '9'
        ≪other settings here≫

config 'wifi-iface'
	option 'device' 'radio0'
	option 'network' 'lan'
	option 'mode' 'ap'
	option 'ssid' 'AAAAAAAA'
	option 'wds' '1'
	option 'short_preamble' '1'
	option 'bssid' 'yy:yy:yy:yy:yy:yy'
```




### WRT54G Router (WRT54G, OpenWRT 10.03 with legacy 2.4 kernel for broadcom wireless)


Content of /etc/config/wireless:

```plain
config wifi-device  wl0
        option type     broadcom
        option channel  9
        ≪other settings here≫

config wifi-iface
        option device   wl0It would seem that 
        option network  lan
        option mode     ap
        option ssid     LOCALSSID
        option hidden 0
        option short_preamble 1
        
config wifi-iface
        option device   wl0
        option network  lan
        option mode     wds
        option ssid     AAAAAAAA
        option hidden 1
        option bssid xx:xx:xx:xx:xx:xx
        option short_preamble 1
```



### Notes







  * xx:xx:xx:xx:xx:xx is the MAC address of the WR1043ND, and yy:yy:yy:yy:yy:yy the MAC address of the WRT54G


  * Both devices have to be on the same channel


  * On the secondary router (WRT54G) you actually connect on the ssid LOCALSSID but all DHCP, etc. comes from the TP-LINK



Note also you need to ensure that DHCP on the LAN is disabled on the second router!

I am unsure why this configuration has apparently been problematic for some - perhaps it is quite router specific as to whether it works.
