---
author: admin1
comments: true
date: 2012-02-09 10:39:29+00:00
layout: post
slug: openwrt-web-admin-luci-wierdness
title: OpenWRT web admin (LUCI) wierdness
wordpress_id: 231
categories:
- howto
tags:
- openwrt
---

I had a router configured with OpenWRT BackFire (10.03) and had a need to upgrade it to support the web management interface.

So to do this login to a console and:
[crayon lang="bash"]
opkg update  
opkg install luci-light luci-theme-openwrtlight 
/etc/init.d/uhttpd enable 
/etc/init.d/uhttpd start
[/crayon]

However, in my attempt to streamline things and reduce disk usage I managed to only have the 'Essentials' menu, this was missing various important items such as detailed dnsmasq settings (which was the whole point of the exercise.) So I pushed on and installed `luci`, which should have enabled the "Administration" menu but did not.

After much mucking around and browsing through the installed files, I stumbled upon this:
[crayon lang="bash"]
rm /tmp/luci-indexcache 
[/crayon]
after which logging in the "Administration" menu is now present.



