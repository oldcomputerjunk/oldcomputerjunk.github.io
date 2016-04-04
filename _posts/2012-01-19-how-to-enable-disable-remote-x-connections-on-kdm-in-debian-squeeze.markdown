---
author: admin1
comments: true
date: 2012-01-19 11:33:51+00:00
layout: post
slug: how-to-enable-disable-remote-x-connections-on-kdm-in-debian-squeeze
title: How to Enable / Disable remote X connections in Debian Squeeze
wordpress_id: 151
categories:
- howto
- linux
tags:
- debian
- LCA2012
- linux
- network
- Xwindows
---

## Background


Using my AspireOne at LCA2012 I realised I had a hole that really needed to be tightened before using the Most Excellent LCA2012 wireless network instead of a 3G dongle.

One of these is that I was gaining an ipv6 address on the LCA network.  (This was not the hole.)  My 3G only has ipv4 and at I only run ipv4 at home.

After connecting I had both ipv6 and ipv4 addresses, but importantly upon running `netstat -antp` realised I had kdm [1] listening on 6000 wide open - I had it firewalled out on ipv4 but had never setup ip6tables (oops)  

To sort things quickly I subsequently disabled ipv6 [2] but I first killed off my local X/server from accepting connections anyway (you cant be too cautious, in a large crowd of very skilled people, potentially prank-minded, right?  following good advice - ['take precautions' ...](http://linux.conf.au/wiki/index.php/Network))


    
    
    me@xxxxxxx:~$ sudo netstat -lntp
    
    Active Internet connections (only servers)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 0.0.0.0:6000            0.0.0.0:*               LISTEN      13494/X         
    tcp6       0      0 :::6000                 :::*                    LISTEN      13494/X  
    





## Procedure


To disable network connections on port 6000 using kdm:


  1. Edit `/etc/kde4/kdm/kdmrc`


  2. Look for a section `[X-:*-Core]`


  3. If missing or commented out, add the line `ServerArgsLocal=-nolisten tcp`, if it is already there, instead append `-nolisten tcp` to the line starting with  `ServerArgsLocal`



  4. Either reboot, or kill X and restart kdm



Result:

    
    
    me@xxxxxxx:~$ sudo netstat -lntp
    
    Active Internet connections (only servers)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    
    



To _enable_ connections, simply ensure the `-nolisten tcp` arguments are not present.

FWIW I have had to do this on gdm in the past as well.
Instructions for this are actually provided in the Debian Reference Manual, [Chapter 7](http://www.debian.org/doc/manuals/debian-reference/ch07.en.html) (section 7.4.2, tips)

YMMV if you have more than a basic configuration or are running some other variant or distribution.



##### Footnotes





[1] I only use kdm for login, for performance I run openbox
[2] Yes, I know, I should use ipv6, I even agree with all the reasons listed by Julien Goodwin at his excellent [SysAdmin miniconf](http://sysadmin.miniconf.org/presentations12.html#01) talk on why I should use ipv6, but that will need to wait until I get home (and depends on my ISP.)
(And facepalm to self for mixing up selinux with ipv6 before coffee this morning)


