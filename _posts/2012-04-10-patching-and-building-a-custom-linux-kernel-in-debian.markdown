---
author: admin1
comments: false
date: 2012-04-10 12:53:17+00:00
layout: post
slug: patching-and-building-a-custom-linux-kernel-in-debian
title: Patching and Building  a custom Linux Kernel in Debian
wordpress_id: 292
categories:
- linux
tags:
- debian
- kernel
- linux
---

These posts cover a topic which seems to be documented to varying degrees across the net, but nothing quite exactly matched what I wanted to do. In the end this is a result of multiple sources of information and inspiration (and perspiration...)

For some time I had been getting a Kernel fault report popup with irritating regularity. In the end I isolated it to something going wrong with my external Firewire drive after my computer was resuming from suspend (specifically Suspend to RAM.)
In the end chasing this down required working through the following tasks:



	
  1. Disabling the proprietary NVidia driver and activating 'nv' ( I was unable to successfully configure nouveau to work with my particular dual head configuration), so that my kernel was no longer 'TAINTED', which would have led me into a brick wall if I had been required to report a kernel bug.

	
  2. Consistently replicating the fault, which included learning about a bunch of stuff in the Linux /sys filesystem.

	
  3. Finally getting a 3-series kernel to work on Debian Squeeze - it turns out by now 3.2 has been packaged into Debian [backports](http://backports.debian.org/), which gets me past an earlier roadblock with kernel upgraded.  Upgrading to the latest kernel would eliminate if the problem had been resolved (which is was not at least of 3.2.9)

	
  4. Rebuilding the kernel from source - (something I have done this many times before, but it doesn't hurt to recap) and applying the patches needed

	
  5. Re-enabling NVidia - which involved verifying my DKMS setup was still working.



I haven't blogged recently due to various family mini-crises to do with pets, sickness and other issues, as well as extra busyness at work.

As it is getting late this post will conclude with the command line used to build and install my kernel, and I will expand on this in the next post.

```bash
cd /usr/src
cd linux-source-3.2
cp /boot/config-3.2.0-0.bpo.1-amd64 .config
make silentoldconfig
CONCURRENCY_LEVEL=4 time fakeroot make-kpkg \
              --initrd \
              --append-to-version=-mine-preempt-amd64 \
              --revision 1~mine1.00.00 kernel_image kernel_headers
sudo dpkg -i linux-{headers,image}-3.2.9-mine-preempt-amd64_1~mine1.00.00_amd64.deb
```

Things to note:


  * The above will build a kernel using the same configuration as an installed Debian backports 3.2 kernel, assuming the backports kernel an source packages have been installed. There are no changes or patches yet


  * Your user must be in the 'src' group for the make-kpkg command to work as-is.


  * The 3.2 kernel in backports (as of March 2012) was in fact version 3.2.9 although this is not indicated in the Debian version for some reason.




