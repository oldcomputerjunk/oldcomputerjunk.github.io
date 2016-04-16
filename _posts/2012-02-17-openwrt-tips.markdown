---
author: admin1
comments: true
date: 2012-02-17 11:12:16+00:00
layout: post
slug: openwrt-tips
title: OpenWRT tips
wordpress_id: 234
categories:
- howto
tags:
- openwrt
---

## "Undeleting" files from the /rom filesystem...


This post describes in gory detail how to recover files from the ROM image in case you accidentally removed a base package from OpenWRT.
In brief, locate files called 'META_*' on /overlay and edit them to remove deletions from the overlay filesystem.  For detail, read on...

**TL;DR**

When first installed, the OpenWRT image is built into a [squashfs](http://www.squashfs.org) read-only filesystem, mounted at `/rom` and underlying the root (/).  Writes to the root filesystem persist on a [jffs2](http://en.wikipedia.org/wiki/JFFS2) filesystem mounted at `/overlay` and merged at the root using [mini_fo](http://wiki.openwrt.org/doc/techref/filesystems).  Ignoring operations occurring actually on `/rom/` and `/overlay paths`, if a file is overwritten, the file is actually stored in `/overlay` at the same relative location, and mini_fo makes the newer file appear to the system form that point.  Operations in `/overlay/` itself are obviously filtered by mini_fo.  `/tmp`, `/dev`, etc. are all mounted on tmpfs outside of this process.

So how are deleted files handled? It turns out there is a special file made in each directory in `/overlay`, that mini_fo hides from view in the applicable root level directory, that records files on `/rom` that are deleted.

The special files are prefixed with 'META_', for  example: `META_dAfFgHE39ktF3HD2sr`, the content is simply one file per line, two fields, the letter D followed by the deleted file: 
```text
D S96led
```


Undelete files by simply removing the entry from the META_ file.

In the event you remove a package that was in the `/rom` image using ipkg, it is easy enough to get back by finding editing all the special files in `/overlay` and removing the deletion references.

Hint: although there are various to choose from, I always start with the smallest `/rom` image, as that allows easier reconfiguration by adding and later removing packages, and provides a bigger R/W jffs2 partition at /overlay.  If you need to delete a package that was in `/rom`, it doesn't actually give you any extra space on the small SSD...

