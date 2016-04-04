---
author: admin1
comments: false
date: 2012-08-07 14:31:13+00:00
layout: post
slug: using-enigmail-with-thunderbird-icedove-10-from-debian-squeeze-backports
title: Using EnigMail with Thunderbird Icedove 10 from Debian Squeeze Backports
wordpress_id: 333
categories:
- howto
tags:
- debian
- email
- gpg
- thunderbird
---

I keep Firefox I mean Iceweasel up to date in Debian Squeeze by tracking the mozilla.debian.net squeeze-backports apt repository.  However I recently realised I was not maintaining my Thunderbird I mean Icedove email client, which was stuck back at version 3.1.16 via the routine backports.debian.org repository!

This was easily fixed by setting up apt.sources to track the trailing edge 10.x series, such that now:
[crayon lang="bash"]
...$ apt-cache policy icedove
icedove:
  Installed: 10.0.5-1~bpo60+1
  Candidate: 10.0.5-1~bpo60+1
  Version table:
 *** 10.0.5-1~bpo60+1 0
        710 http://mozilla.debian.net/ squeeze-backports/icedove-esr amd64 Packages
        100 /var/lib/dpkg/status
     3.1.16-1~bpo60+1 0
        100 http://backports.debian.org/debian-backports/ squeeze-backports/main amd64 Packages
[/crayon]

However, I discovered Enigmail, which integrates OpenPGP into Thunderbird, was now broken.  I was getting the following message: 


<blockquote>**Unable to locate GnuPG executable in the PATH**</blockquote>


This was also visible in the Enigmail Addon Preferences dialog box.  This error stubbornly refused to go away even if I entered the full path to /usr/bin/gpg - after a search and perusal of several forums and mailing lists it soon appeared this is an error which is infamous for having an elusive solution.

First realising I had Enigmail installed from Debian stable, I uninstalled via apt the enigmail package.  This was after unsuccessfully attempting removal via the Addon manager, which failed no doubt because as a Debian package the addon was installed in a different location with different filesystem privileges.

However, subsequently installing Engimail via the Addon Manager download XPI also failed to resolve the problem.

Eventually I resolved the problem by making a local source build of Enigmail 1.4 from sources downloaded from Debian Wheezy (testing)  Along the way I had to deal with some minor dependency issues: apt-get refused to pick out the correct version in some cases until manually overridden; I also had a long forgotten .mozconfig file in my home directory which conflicted the build script until I removed it.

[crayon lang="bash"]
apt-get source enigmail # <-- my configuration pulls this from wheezy (testing) by default
sudo apt-get install mozilla-devscripts  libdbus-glib-1-dev libnotify-dev autoconf2.13
sudo apt-get install libnss3-1d libnss3-dev=2:3.13.5-1~bpo60+1 icedove-dev
cd enigmail-1.4.1
dpkg-buildpackage -rfakeroot -b
cd ..
sudo dpkg -i enigmail_1.4.1-2_amd64.deb
[/crayon]

Restart Icedove, and all is now good again.

