---
author: admin1
comments: false
date: 2013-06-30 12:05:34+00:00
layout: post
slug: debugging-an-x86-bootloader-using-qemukvm
title: Debugging an x86 bootloader using QEMU+KVM
wordpress_id: 426
---

I have a disk image of a Windows 2000(!) computer I saved some years ago, and I wanted to run it up to access some software/data saved within.  

Now I could of course just loopback mount the NTFS image and access the files, and I was able to successfully do this.  However the data I want is unfortunately in a form that I need to export from the software that created it.  This requires running up the Windows system.

I have enough hardware stored (hoarded?) to be able to build a suitable system, but if anything was a candidate for virtualisation, this would be it, right? 

**TL;DR** lots of detail follows...

Now just transferring a Windows 2000/XP drive between systems can be a headache enough.  Especially when the CPU or motherboard chipsets, are different... in the past this has involved changing hal.dll , editing the registry and removing chipset specific IDE drivers, and of course VGA drivers, etc, etc.  I hoped that running in a VM using 'common' devices (i440FX, Cirrus VGA) would alleviate some of this.

Before even getting that far though, my disk image was actually a NTFS partition image.  Meaning work is needed before even attempting to boot into it.  Typically for systems of that era, I expected the NTFS partition would have started at logical sector 63, (or Cylinder 0, Head 1, Sector 1) - and I found myself dealing with Cylinder-Head-Sector (CHS) addresses for the first time in rather a few years!  To get the image to boot I had to construct a boot sector and additional sectors, edit the Master Boot Record partition table to point to the partition and concatenate the partition.

In all cases I was working with a copy of the original image.  One cant be too careful.

For maximum compatibility I started by purloining a [MBR](https://en.wikipedia.org/wiki/Master_boot_record) sector from another Windows 2000 install.
The partition record for the first primary partition starts at hex byte 0x1be in sector 0.  Of particular interest is the partition starting sector, the ending sector in CHS, and logical sectors:
 

    
    0001b0 00 00 00 00 00 2c 44 63 74 a5 94 9a 00 00 80 01  >.....,Dct.......<
    0001c0 01 00 07 fe ff fd 3f 00 00 00 7e c5 7c 00 00 00  >......?...~.|...<
    0001d0 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
    



Here, 0x1bf contains the starting head, 0x1c0 bits 0-5 are the sector and 0x1c1 and the upper two bits of 0x1c0 comprise the cylinder.  Thus, we have C=0,H=1,S=1, which in logical terms is 63.

Aside: the formula for translation is as follows:
LogicalAddress = (NS x NH) x C + (NS x H) + (S-1)  where NS = sectors/track and NH = heads/disk
Nominally, as reported by Linux fdisk when executing an arbitrary Xubuntu image inside kvm, we can start by selecting NS=63 and NH=255.  Therefore when C/H/S = 0/1/1 --> (0) + (63x1) + (1-1) == 63.  Note that sectors are numbered in 1..63 but cylinders and heads in 0..x ... sectors count up in sector, head, cylinder order e.g. 0/0/1..0/0/63,0/1/1..0/1/63,0/2/1..0/2/63,0/254/1..0/254/63,1/0/1..1/0/63, etc.

Aside: as I discovered later, the C/H/S size of a disk reported by fdisk can be somewhat arbitrary and vary from the preferred or physical reality.  Let alone what the combination of virtual machine host/ / BIOS decide to report it as...

The byte at 0x1c2 is 0x07 which indicates this is an NTFS partition.

The three bytes at 0x1c3 are the CHS address of the last sector. 
The four bytes at 0x1c6 is the logical address of the first sector, which is 63
The four bytes at 0x1ca is the size of the partition in sectors.
I had to edit the bytes between 0x1c3 and 0x1cd.

Now my partition image was 3923426304 bytes or 7662942 sectors (sectors being 512 bytes in this case.)
So I had to replace bytes 0x1ca..0x1cd with 0x5e 0xed 0x74 0x00 which is 0x74ed5d or 7662942 in 32-bit in least significant byte (LSB) first ordering.
The tricker bit is in sorting out the C/H/S equivalent.  Using what fdisk reported, I needed to work out the end C/H/S address.  This would correspond to logical address 63 + 7662942 - 1 (as logical addresses are zero based.)  To start with, this is sector 7663004.  7663004 / (63*255) = 476.99..., so our end cylinder is 476.  We take the remainder, 7663004 - (476*63*255), or 16064, and divide by our number of sectors, 63.  This yields 254 remainder 62.  Thus our ending address in C/H/S terms is 476/254/62, except that sector numbers are 1-based, so we need to write 476/254/63 into bytes 0x1c3.

Modified partition entry:
   

    
    0001b0                                           80 01
    0001c0 01 00 07 fe 7f dc 3f 00 00 00 5e ed 74 00
    



One thing to take away here is that this legacy partition record in most cases uses the absolute address, because C/H/S addressing in 3 bytes limits us to the first 8GB of the disk (this is a well known limitation in older versions of various operating systems.)

Note also, as the logical address is 32 bits, this is where the 2TB limit of the legacy MBR comes from, and why any drive > 2TB needs to use a [GPT](http://en.wikipedia.org/wiki/GUID_Partition_Table) partition table to access more than 2TB!

The boot sector was a 512 byte file.  I edited the easy way using ghex2.  I still needed to fill 62 sectors before the NTFS partition. Then join it all together.

```bash
~$ dd of=padding.bin bs=512 count=62 if=/dev/zero
~$ cat mbr.bin padding.bin ntfs_disk.img > test.img
```

This resulted in a file I could use with kvm.

The acid test was seeing if the geometry edits worked.  I booted the KVM guest with xubuntu, and was successfully able to mount and copy files from my appended partition.

As it turns out that was the easy bit.

When I attempted to boot into the NTFS image, I was greeted with "A disk read error occurred." - not very hopeful.  At this stage I wasn't sure if I had edited the MBR incorrectly, or the problem was with KVM/QEMU.  The usual error message I was used to seeing with Windows was "NTLDR missing."  After a bit of trial and error I decided to step through the bootsector with the debugger, to see if it was chaining to Windows properly.  (What geek wouldn't pass up such a chance :-)  of course, later confirmed with hex editor, the string 'A disk read error occurred' is located in the NTFS boot file.  Which would have saved some effort.  But then I wouldn't be writing about how to debug the bootloader, or learning about debugging with QEMU, or  applying x86 disassembler skills I hadn't used for rather a while...)

Procedure:



	
  1. Start the virtual machine, and make it stop for single stepping after starting the CPU. The important options here are `-s -S`.
```bash
~$ ~/opt/qemu-1.4.2/bin/qemu-system-i386 -boot c -m 256 -enable-kvm -hda disk.img -cpu 486 -no-acpi -s -S
```


	
  2. Start gdb and remotely attach to the VM.
```bash
~$ gdb
target remote localhost:1234
set architecture i8086
display /i ($cs*16)+$pc
stepi  # step into the guest BIOS (seabios)
stepi
stepi
stepi
stepi
stepi
stepi
stepi
stepi
stepi
stepi
br *0x7c00 # set breakpoint in the MBR
cont
```




Now, the multiple `stepi` instructions I found I needed to get this to work at all.  In theory, according to what documentation I found, after 'cont' is executed the debugger should have stopped the VM at the start of the MBR (0x7c00), however there seems to be a timing glitch between KVM and gdb that caused this breakpoint to be ignored.  Upgrading to the latest kvm and gdb didn't help, but by fluke I managed to stop it and realised I had stepped several instructions into the BIOS first.

To streamline this process, which I repeated a few times, I made a gdb script and executed it using `gdb -x script.txt` - the script file contains the commands up to 'cont' above.

To assist this process, I replaced the MBR code with the code supplied by Debian in the ['mbr'](http://packages.debian.org/squeeze/mbr‎) package, so that I had source code to follow along with.  And I soon discovered, that the geometry was processed correctly, and the problem was indeed Windows.

After a little more detective work, I was able to confirm (as suspected) that the Windows NTFS bootloader was using ['int13'](http://en.wikipedia.org/wiki/INT_13H) calls, and it was unhappy about the fact that I was using heads > 16.  Wierd.  However, I recalculated the C/H/S addresses assuming 16 heads instead of 254, and shortly thereafter, I was confronted with the weirdly comforting 'NTLDR missing' error message!

At this point it was getting (very) late, so I stopped.
