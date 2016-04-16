---
author: admin1
comments: false
date: 2013-11-19 11:21:59+00:00
layout: post
slug: quick-tip-copying-colons-over-ssh-and-rsync
title: Quick Tip - copying colons over ssh and rsync
wordpress_id: 457
categories:
- tech
tags:
- ssh
---

If you happen to want to copy a file in the current directory  with a colon in the name:
```bash
scp some-file:123.txt user@host:/path/to/some-place```
this will fail.  Possibly after quite a timeout, with a completely unrelated error (unresolved domain name some-file maybe?)

This is because ssh uses colons to separate the user@host part from the filename part.

The fix when the source is on the local computer is to ensure the path starts with a dot:
```bash
scp ./some-file:123.txt user@host:/path/to/some-place```

Incidentally this is related to that old newby fail, forgetting the trailing colon when coping to a destination home directory:
```bash
scp some-file.txt user@host```
results in a file in the current directory called 'user@host'. Oops!

