---
author: admin1
comments: false
date: 2012-11-24 10:43:17+00:00
layout: post
slug: when-ssh-x-somehost-doesnt-work
title: When 'ssh -X somehost' doesnt work...
wordpress_id: 382
categories:
- howto
- tech
tags:
- ssh
---

So I went to try and remote run an xterm from a machine I had not turned on for a while...
[crayon lang="bash"]
~$ ssh -X me@somehost 
... 
~$ xterm
[/crayon]
And unexpectedly received the following error message:
[crayon highlight="false"]
xterm Xt error: Can't open display: 
xterm:  DISPLAY is not set
[/crayon]
After some scratching around I found the following error message in `/var/log/auth:`
[crayon highlight="false"]
sshd  error: Failed to allocate internet-domain X11 display socket.
[/crayon]
After more had scratching and reading the `sshd_config` [manpage](http://linux.die.net/man/5/sshd_config),  I remembered I often turn off ipv6 on test machines on my local network (yes, I know...)  This doesn't always play well with default configs of a lot of software, including ssh. Enabling the following setting in `/etc/ssh/sshd_config` resolved the problem:
[crayon highlight="false"]
AddressFamily inet
[/crayon]

