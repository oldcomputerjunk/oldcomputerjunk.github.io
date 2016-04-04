---
author: admin1
comments: true
date: 2012-01-02 06:29:39+00:00
layout: post
slug: building-gnucash-from-getdeb-sources-and-passing-configure-options-to-dpkg-buildpackage
title: GnuCash 2.4.8 on Debian Squeeze
wordpress_id: 61
categories:
- howto
- linux
tags:
- configure
- debian
- dpkg
- gnucash
- python
---


I like to retain for as long as possible the ability to install packages into my Debian system (currently, 'squeeze') from stable DVD ISO images made from physical disks previously purchased.  This means that at times I have to do some dpkg-buildpackge / make / configure trickery to get things t work as I would like...


<!-- more -->



The reason for maintaining a stable distribution local package archive are twofold:




  * out here our Internet is wireless as it is in much of his country outside the cities, and thus both bandwidth and download-limit constrained.


  * I can track Debian security updates for the most part.


Of course, this means when I want to run newer software I frequently need to rebuild from source.  Simply downloading binaries from wheezy or the latest Ubuntu would otherwise want to upgrade much of my system to a newer libc6, etc.  I also like to try and keep the my installed software 'debianised' as much as possible by building the DEB files from sources, so I don't end up with much crap in `/usr/local/bin` or other places.  



Recently I wanted to upgrade [GnuCash](http://www.gnucash.org) from 2.4.5 (previously built from source) to 2.4.8 as found in wheezy.  I want to experiment with the python bindings to better customise my own budget reporting, as well as hopefully fix some annoying UI bugs.



The usual method for building a package `dpkg-buildpackage -rfakeroot -b` doesn't suffice in this case, due to missing dependencies (libAqbanking is too new, for example, and in any case I don't do banking from within GnuCash) so I had to go hunting for documentation (see references below)





### Complete procedure:







  1. 
Download / install the prerequisites (note - this may vary depending on what is already installed).
This assumes that wheezy is also in `sources.list` as some of these install from wheezy with no dependency issues.
[crayon lang="bash"]
sudo aptitude update
sudo aptitude install libwebkit-dev python-dev libdbi-dev
sudo aptitude install dh-autoreconf flex
sudo aptitude install libgmp-dev -t wheezy
[/crayon]



  2. 
Build GnuCash deb package 2.4.8 from source, the follow method will ignore dependencies on libAqBanking (and build without that feature):
[crayon lang="bash"]
apt-get source guile-1.8 -twheezy
apt-get source gnucash -twheezy
export MAKEFLAGS=-j5
cd guile-1.8
# Hack package by stopping tests being run - on my system I have ipv6 disabled atm which caused the self-test to fail?
echo 'override_dh_auto_test:' >> debian/rules
dpkg-buildpackage -rfakeroot -b
cd ..
sudo dpkg -i guile-1.8_*.deb guile-1.8-libs*.deb
cd gnucash-2.4.8
DEB_BUILD_OPTIONS="parallel=5" fakeroot debian/rules binary
cd ..
sudo dpkg -i gnucash*.deb python-gnucash*.deb
[/crayon]






Note that the option `parallel=5` should in theory invoke `make -j5` to enable use of multiple cores to speed up the build. In practice this may not happen due to possible bugs in packaging so it may pay to `export MAKEFLAGS=-j5` as well...





### Things I discovered along the way


Gnucash (and probably many other packages) seem to ignore the recommended use of `DEB_BUILD_OPTIONS` as a mechanism for passing additional options through to configure, which is disappointing



References:




  * 
[How to: Recompiling / Rebuild Debian / Ubuntu Linux Binary Source File Packages](http://www.cyberciti.biz/faq/rebuilding-ubuntu-debian-linux-binary-package/)



  * 
[HowTo Build a Package from Source the Smart Way](http://forums.debian.net/viewtopic.php?p=228570)



  * 
[Debian New Maintainers' Guide Chapter 6. Building the package](http://www.debian.org/doc/manuals/maint-guide/build.en.html)



  * 
[Debian Policy Manual
Chapter 4 - Source packages ](http://www.debian.org/doc/debian-policy/ch-source.html)







