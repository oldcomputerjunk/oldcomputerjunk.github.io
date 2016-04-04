---
author: admin1
comments: true
date: 2012-01-22 12:33:03+00:00
layout: post
slug: remastering-youtube-videos-into-avi-for-your-set-top-box
title: Remastering Youtube videos into AVI for your Set Top Box
wordpress_id: 193
categories:
- howto
- tech
tags:
- debian
- github
- LCA2012
- linux
- youtube
---

I wasn't going to post here for a week or so as I need to clear my head and catch up with real world stuff, but I did this and thought it would be helpful to share.

As of now most of the [linux.conf.au](http://linux.conf.au) sessions have been uploaded onto YouTube.  All the LCA2012 videos can be found at [www.youtube.com/user/linuxconfau2012/videos](http://www.youtube.com/user/linuxconfau2012/videos).  There is speculation (most likely) they will be mirrored at some point but I don't want to wait so I have started watching them from YouTube.

<!-- more -->
On Linux you can download YouTube videos for later viewing using the [clive](http://clive.sourceforge.net) program.  This is packaged in Debian, YMMV for other distributions.  Clive works like wget in that you can simply copy and paste the YouTube URL onto the command line and grab the [FLV](http://en.wikipedia.org/wiki/Flv) file.  The file can be played on your PC or moved onto a smart phone or media player device.

I tend to use my PC as a PVR and play recordings on my set top box by USB key.  Unfortunately the FLV  files cannot be played directly on my STB, which is a cheap Dick Smith model, so need to be converted first.



### Procedure




  1. If required, install clive and mencoder.  On Debian, this is done using the following command:
[crayon lang="bash"]
sudo apt-get install clive mencoder
[/crayon]



  2. Navigate to the YouTube video page, and copy the URL.  Then in a terminal, enter the following command, pasting the URL where indicated.  Note that the single quotes are important as a couple of times I found that the URLs have characters that have meaning to bash:
[crayon lang="bash"]
clive 'Paste the URL here'
[/crayon]



  3. After downloading completes, remaster the file into AVI format.
[crayon lang="bash"]
mencoder somefile.flv -oac mp3lame -lameopts cbr:br=64 -ovc xvid \
  -xvidencopts bvhq=1:quant_type=mpeg:bitrate=1300:pass=2:turbo:threads=3 \
  -o somefile.avi
[/crayon]



Copy the resulting file onto your USB stick and enjoy in your loungeroom :-) This is good for showing them to digitally challenged friends and family as well...

If the video looks stretched, see below.

You may omit the backslash above and type the commands with a backslash all on one line.

I will trying to show to quite a few people [Karen Sandlers](http://linux.conf.au/media/news/51) keynote [ video](http://www.youtube.com/watch?v=5XDTQLa3NjE&list=PL98382D6677F8E2D4&index=1&feature=plpp_video), which to me is about 'the Freedom to Stay Alive', and shoving a USB into someones STB or Windows laptop can often be more effective than even emailing a YouTube link.

PS. kudos to the [Crayon](http://wordpress.org/extend/plugins/crayon-syntax-highlighter/) syntax highlighter plugin for Wordpress.

**TL; DR**

You could probably go further, and write a scraper to do all the videos and encoding in one fell swoop but I don't have time so I have been cherry picking what I want to see soonest or show others, first.



### Caveat


My system is not a pure Debian Squeeze installation, I have quite a number of repositories in my `/etc/apt/sources.list`.  My version of mencoder actually comes from [www.debian-multimedia.org](http://www.debian-multimedia.org) and it is possible that the pure Debian version may not perform identically... my system has also been configured for some time so there may be other dependencies not listed above, again, YMMV.

The threads=3 argument is for my quad core Phenom, so you may need to reduce this for a dual core processor.

I saved the command into a script:
[crayon lang="bash"]
#!/bin/bash
mencoder "${1}" -oac mp3lame -lameopts cbr:br=64 \
  -vf scale=512:376,expand=648:384 -ovc xvid \
  -xvidencopts bvhq=1:quant_type=mpeg:bitrate=1300:pass=2:turbo:threads=3 \
  -o `basename "$1"`.avi
[/crayon]
Then you can run it in a loop over all videos:
[crayon lang="bash"]
./flv2avi.sh *.flv
[/crayon]

The conference video is recorded at 320x240 which is a 4:3 aspect ratio.  This means by default it plays fine on older televisions, but will often be stretched by default on modern televisions with a 16:9 aspect ratio.  This can be centered usually with a button press on the remote, but sometimes even this is beyond, umm, less experienced friends and family (like elderly relatives) so you can instead 'pillar-box' the video into 16:9 so that it doesn't need reducing - although this will cause it to look compressed if played on a 4:3 television:

[crayon lang="bash"]
mencoder somefile.flv -oac mp3lame -lameopts cbr:br=64 \
  -vf scale=512:376,expand=648:376 -ovc xvid \
  -xvidencopts bvhq=1:quant_type=mpeg:bitrate=1200:pass=2:turbo:threads=3 \
  -o somefile.avi
[/crayon]

