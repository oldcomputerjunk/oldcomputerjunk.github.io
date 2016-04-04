---
author: admin1
comments: false
date: 2012-11-13 11:58:10+00:00
layout: post
slug: fixing-sluggish-write-performance-of-usb-flash-thumb-drives
title: Fixing sluggish write performance of USB flash (thumb) drives
wordpress_id: 364
categories:
- linux
- tech
tags:
- disks
- linux
- performance
- usb
---


This has been noted in various places around the web but in practice what I did seems to be a combination of various writings so I have documented my own experiences here.



## Background



I recently acquired  a (yet another) USB flash drive, this a 16 GB "Dolphin" brand.  The actual device reports as "048d:1165 Integrated Technology Express, Inc." when interrogated using lsusb.  I am using it to transfer transcoded [Kaffeine](http://kaffeine.kde.org/) PVR recordings from my PC to the set top box in the lounge for more comfortable watching.  

On first use, however, it took what seemed like forever to transfer a 250MB AVI file, over USB2, and looking at the [GKrellM](http://en.wikipedia.org/wiki/GKrellM) chart the write data rate appeared to be a very poor 350 kB/sec.  So it seemed yet again, I needed to optimise a USB disk before it was adequate for use.

In theory, to simplify things to one sentence, flash disk (and in particular, modern SSD) should be faster than spinning disks, as access is a true physical random access operation, without having to wait for the heads to be in the right spot.  However this is invalidated due the blocky nature of flash disk writes.  The actual reason for the poor write speed is that the default partition starts at the 63rd sector (byte 32256) on the disk, and USB flash drives, SD cards, etc. are designed to write data in chunks of say 128kB at a time.  Even if you only write one sector, the entire 128kB (or 256 sectors) must be (re-read first and) written.  So when a partition is not aligned on a 128kB boundary, more writes than otherwise necessary are required, slowing performance.  USB flash drives generally employ FAT32 so they are usable on the widest variety of devices (including set top boxes) and the general experience of FAT32 is that write performance is severely affected if the partition alignment does not match the flash write size, for both the partition and the FAT master table itself.



## Procedure



The procedure I follow for doing fixing misaligned flash drives is:

        
    1. Find a Linux computer, or reboot using a live Linux distribution such as [SysRescueCD](http://www.sysresccd.org/SystemRescueCd_Homepage)
	
    2. Destroy the existing partition.

	
    3. Recreate a single partition, ensuring it starts at the 256th sector (byte 131072, or 128kB)

        
    4. Format the partition to FAT32. with the following non default options:

      * override the default sectors per cluster to ensure clusters are aligned.  This comes at some expense of apparent usable space, but the performance gain for writing large files such as video files is more than worth it.
      * Adjust the "reserved" sectors so that the FAT table itself is aligned to 128kB.

        





## Detailed Steps



The following command sequence will accomplish this under Linux.  This assumes your drive is at /dev/sdd, this will vary depending on what other disks you have.


    1. Run GNU fdisk with units in sector mode not cylinder mode.  Then print the existing partition table (enter _p_ when prompted.  Below you can see the start sector of the existing partition is at sector 63.  Note this is also a primary partition.  This is typical of USB flash disks you might purchase at the local supermarket...
[crayon lang="bash"]fdisk -u /dev/sdd[/crayon]
[crayon highlight="false"]
GNU Fdisk 1.2.4
Copyright (C) 1998 - 2006 Free Software Foundation, Inc.
This program is free software, covered by the GNU General Public License.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

Using /dev/sdd
Command (m for help): p                                                   

Disk /dev/sdd: 16 GB, 16162675200 bytes
255 heads, 63 sectors/track, 1965 cylinders, total 31567725 sectors
Units = sectors of 1 * 512 = 512 bytes

   Device Boot      Start         End      Blocks   Id  System 
/dev/sdd1              63    31570943    15791863    c  FAT32 LBA
Warning: Partition 1 does not end on cylinder boundary.                   
Command (m for help):   

[/crayon]



    2. Delete the partition:
[crayon highlight="false"]
Command (m for help): d
Partition number (1-1): 1                                                 
Command (m for help): p

Disk /dev/sdd: 16 GB, 16162675200 bytes
255 heads, 63 sectors/track, 1965 cylinders, total 31567725 sectors
Units = sectors of 1 * 512 = 512 bytes

   Device Boot      Start         End      Blocks   Id  System 
Command (m for help): 
[/crayon]



    3. Recreate the partition, aligned at sector 256 (131072 bytes), and set the type back to FAT32 LBA (in this case matching what previously existed) (type 'c', or 0x0c, i.e. FAT32 LBA).  Use of FAT32 LBA allows use to start the filesystem on an arbitrary sector bearing no relationship to legacy cylinders, etc.  The final sector depends on the disk size.
[crayon highlight="false"]
Command (m for help): n                                                   
Partition type                                                            
   e   extended
   p   primary partition (1-4)
p
First sector  (default 63s): 256s                                         
Last sector or +size or +sizeMB or +sizeKB  (default 31567724s):          
Command (m for help): t                                                   
Partition number (1-1): 1                                                 
Hex code (type L to list codes): c                                        
Changed type of partition 1 to c (FAT32 LBA)
Command (m for help): p                                                   

Disk /dev/sdd: 16 GB, 16162675200 bytes
255 heads, 63 sectors/track, 1965 cylinders, total 31567725 sectors
Units = sectors of 1 * 512 = 512 bytes

   Device Boot      Start         End      Blocks   Id  System 
/dev/sdd1             256    31567724    15783831    c  FAT32 LBA
Command (m for help):   
[/crayon]



    4. Save changes:
[crayon highlight="false"]
Command (m for help): w                                                   
Information: Don't forget to update /etc/fstab, if necessary.             


Writing all changes to /dev/sdd.
[/crayon]



    5. Format the partition, setting the number of reserved sectors so that the FAT table remains aligned at a 128kB boundary.  Assuming sectors per cluster, s=128 (65536 bytes), and our partition length of 31567469 sectors, we want the first fat to start at the 256th sector within the partition (which is OK as the partition itself is aligned.)  For some sizes of flash disk, this can be an iterative process, but generally setting the number of reserved sectors to 256 will achieve what we want.
[crayon lang="sh"]
mkfs.vfat -v -F 32 -n label -s 128 -R 256 /dev/sdd1
[/crayon]
[crayon highlight="false"]
mkfs.vfat 3.0.9 (31 Jan 2010)
/dev/sdi1 has 255 heads and 63 sectors per track,
logical sector size is 512,
using 0xf8 media descriptor, with 31567468 sectors;
file system has 2 32-bit FATs and 128 sectors per cluster.
FAT size is 2048 sectors, and provides 246586 clusters.
There are 256 reserved sectors.
Volume ID is 3bd81e55, volume label fatflash   

[/crayon]



    6. This is the most important step - verify that the chosen number of reserved sectors has resulted in an aligned FAT table and aligned data area.
[crayon lang="sh"]
fsck.vfat /dev/sdd1
[/crayon]
[crayon highlight="false"]
fsck from util-linux-ng 2.17.2
dosfsck 3.0.9 (31 Jan 2010)
dosfsck 3.0.9, 31 Jan 2010, FAT32, LFN
Checking we can access the last sector of the filesystem
Boot sector contents:
System ID "mkdosfs"
Media byte 0xf8 (hard disk)
       512 bytes per logical sector
     65536 bytes per cluster
       256 reserved sectors
First FAT starts at byte 131072 (sector 256)
         2 FATs, 32 bit entries
   1048576 bytes per FAT (= 2048 sectors)
Root directory start at cluster 2 (arbitrary size)
Data area starts at byte 2228224 (sector 4352)
    246586 data clusters (16160260096 bytes)
63 sectors/track, 255 heads
         0 hidden sectors
  31567468 sectors total
Checking for unused clusters.
Checking free cluster summary.
/dev/sdd1: 1 files, 1/246586 clusters
[/crayon]
The important figure here, is the data area sector - it must be an integer multiple of 256, and 256 x 17 == 4362 in this example.



    7. Test the result.  I copied a 256 MB file onto the drive, and GKrellM is now reporting ~2.5MB/sec.  More importantly, it finished in approx. one eighth of the time compared to before reformatting.



The improved write performance should be just as noticeable from Windows.



