---
author: admin1
comments: true
date: 2012-01-06 08:54:21+00:00
layout: post
slug: using-sshfs-with-a-specific-identity
title: Using SSHFS with a specific identity
wordpress_id: 67
categories:
- howto
- linux
tags:
- blog
- fuse
- ssh
---

The program [sshfs](http://fuse.sourceforge.net/sshfs.html) is a Linux tool which lets you mount a remote directory accessible via the SSH protocl as a filesystem over [fuse](http://en.wikipedia.org/wiki/Filesystem_in_Userspace).  However it defaults to using 'id_rsa' as the SSH identity and there are no direct command line options for selecting an alternate identity, or for that matter other SSH parameters such as the TCP port to connect to.
<!-- more -->
The reason for getting sshfs working with additional options was I wanted to try executing some detailed file and archiving operations in my web space, and I wanted to script it locally from my dev machine.  SSH access required both a different TCP port and a non-default identity.

The solution:
[crayon lang="bash"]
sshfs -o ssh_command="ssh -i ~/.ssh/somekey -p 11234" \
        user@destination.host: localMountPoint
[/crayon]

For the above to work, your local account also needs to be in the 'fuse' group (You may need to logout for this to take effect):
[crayon lang="bash"]
sudo groupmod -a -G fuse username
[/crayon]

References:



  * [problem doing sshfs mount with public key authentication](http://ubuntuforums.org/showthread.php?t=829066).


  * [sshfs(1): - filesystem client based on ssh - Linux man page](http://linux.die.net/man/1/sshfs)


  * [ssh(1): OpenSSH SSH client - Linux man page](http://linux.die.net/man/1/ssh)




