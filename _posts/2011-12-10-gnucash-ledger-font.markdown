---
author: admin1
comments: true
date: 2011-12-10 17:13:15+00:00
layout: post
slug: gnucash-ledger-font
title: Gnucash ledger font
wordpress_id: 28
categories:
- howto
- linux
tags:
- gnucash
- linux
---

I use [GnuCash](http://www.gnucash.org) to track our family finances, and find when I leave it a while to update my credit card spending, etc. I am forever scrolling up and down in the window. By default there are not that many lines displayed on my screen, so I decided to try and change the font.




Unfortunately there are no font options (other than for "checks" [1])...
Eventually I found this [post](http://lists.gnucash.org/pipermail/gnucash-devel/2005-November/014915.html) in the Gnucash mailing list archive.




So I created a new file `~/.gtkrc-2.0.gnucash` and added the following
```bash
style "gnc-register"
{
  font_name    = "Sans 8"
}
widget "*.GnucashSheet" style "gnc-register"
```
Restart Gnucash and voila a full screen of entries.





These settings were tested on GnuCash 2.4.5 on Debian Squeeze.  If you are using Gnucash on Windows your settings will be somewhere else...


  


[1] Here in Australia these are properly spelled as Cheques
