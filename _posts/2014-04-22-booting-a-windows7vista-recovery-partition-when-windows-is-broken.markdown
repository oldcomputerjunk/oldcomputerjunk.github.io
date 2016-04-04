---
author: admin1
comments: false
date: 2014-04-22 12:09:07+00:00
layout: post
slug: booting-a-windows7vista-recovery-partition-when-windows-is-broken
title: Booting a Windows7/Vista Recovery partition when Windows is broken
wordpress_id: 513
categories:
- howto
- windows
tags:
- boot
- grub
- microsoft
- vista
---

Even though I am "pretty much" an open source advocate, I still have to use Windows professionally when required, and of course am the IT support for extended family :-) In this case, I needed to rebuild a laptop for my mum from scratch.

The laptop in question, a Benq Joybook A52 had previously been my dads, and been through incarnations of Windows Vista, downgraded to XP then back up to Vista, and I decided it would be safer to start with a clean slate and perform a factory restore, apply the service packs and all the recent security updates anew.

This laptop had been previously been cleansed of the usual 'crapware' and other default programs, including the factory recovery icon.  I had early on installed the very useful tool EasyBCD from http://neosmart.net/EasyBCD/ to dual boot Vista and XP.  (Aside: EasyBCD used to be free, it seems you can still get it free for Non-Commercial use but you have to dig a bit.)  Using a Linux bootable USB I was able to detect that the recovery partition, a FAT32 primary partition labelled 'PQSERVICE' but set to type 0xde (Dell Utility) was still present (luckily).  However regardless of which settings I tried I was unable to immediately get the factory recovery partition to start, either from the rescue USB or via  EasyBCD.  Being an mum & dads for dinner I didn't have a lot of time to get into nuts and bolts.

Surprisingly, a quick search for PQSERVICE, booting benq recovery partition or various other combinations didn't really bring up anything that useful.  So for the moment /dev/sda4 remained stubbornly inaccessible to the Windows boot machinery.

A couple of weeks later, now having the laptop in my possession to deal with this, I took an image so that I could experiment.  Luckily the drive was only 80GB, such an expansive size from circa 2006!

First thing I did was create an image to play with in qemu.  Figuring that the important parts were simply the recovery partition and the boot sector, I managed this as follows:

(Aside - these instructions may or may not also work on other flavours of laptop of this vintage!)

Firstly, upon mounting the recovery partition, you can see what appears to be a bog standard cut down Windows filesystem:

[crayon lang="plain"]
total 312
drwxr-xr-x  3 root root   4096 Mar 27  2007 Documents and Settings
drwxr-xr-x 11 root root   4096 Mar 27  2007 MiniNT
-rwxr-xr-x  1 root root  47564 Feb 28  2006 NTDETECT.COM
-rwxr-xr-x  1 root root 260272 Feb 28  2006 NTLDR
[/crayon]

Examine the partition layout:

[crayon lang="bash"]
parted laptop.img -s unit s print
[/crayon]

Results:
[crayon lang="plain"]
 1      63s         57401969s   57401907s  primary   ntfs         boot
 2      57403392s   86075391s   28672000s  primary   ntfs
 3      86075392s   139909119s  53833728s  extended               lba
 5      86077440s   139909119s  53831680s  logical   ntfs
 4      139910085s  156296384s  16386300s  primary   fat32        diag
[/crayon]

The recovery partition is #4.  Note that 'diag' is actually type 0xde when checked using fdisk.

Second, assemble a fresh experimental disk:

[crayon lang="bash"]
dd if=laptop.img bs=512 count=63 > bootsectors.bin
dd if=laptop.img bs=512 skip=139910085 > recovery.bin
cp bootsectors.bin test.bin
# now, set the size to match the start of the recovery partition
# luckily it is at the end of the disk
truncate -s $(( 139910085 * 512 )) test.bin 
cat recovery.bin >> test.bin
[/crayon]

As a check the size of test.bin and laptop.img should be identical.

Now, attempt to boot the image in QEMU.

This is achievable using Grub2, and in this case I chose SuperGub2Disk, [http://www.supergrubdisk.org](http://www.supergrubdisk.org).

[crayon lang="bash"]
qemu -enable-kvm  -hda test.bin -m 1024 -cdrom super_grub2_disk_i386_pc_2.00s2-rc5.iso -boot d
[/crayon]

After the boot screen starts, choose 'c' for a command line, and use the following:
[crayon lang="plain"]
set root=(hd0,4)
insmod ntldr
ntldr ($${root})/ntldr
boot
[/crayon]

For once, this worked first time, starting the Powerquest recovery software.

So now I simply had to repeat this process, using a USB key with Grub2 on the laptop itself.


_Prologue_

For some reason the BIOS in this laptop did not understand my favourite bootable USB with Grub2 so I ended up burning a CDR for the first time in a little while.

I also had to wipe the other partitions out of the partition table; it seems the recovery program just unpacks a partition to C: rather than rebuilding the partition table! Things to note include, remember to set the partition type of what will become C: (/dev/sda1) as bootable and NTFS; and for good measure zero the first sectors of that partition.
