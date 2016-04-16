---
author: admin1
comments: false
date: 2012-05-01 05:33:26+00:00
layout: post
slug: fixing-my-annoying-kernel-bugs-part-1
title: Fixing my annoying kernel bug(s) - Part 1
wordpress_id: 298
categories:
- linux
tags:
- kernel
---

This blog entry details some of the problem outlined in [this post.](/2012/patching-and-building-a-custom-linux-kernel-in-debian/) 

Regularly enough to almost be annoying, I was having a kernel fault popup (see stack trace following this blog.)  This was not quite annoying enough to do something about for a long time because the computer wasn't crashed and there were no obvious side effects.  Eventually however I realised that each time it occurred a new instance of my external backup drive was being mounted automagically, so being a little cautious about potential data loss decided to try and get to the bottom of things.

After a few days taking notes and some experimentation I discovered the following:




  * It would happen with regularity after waking the computer up from suspend to RAM.


  * I could force it to happen by 'ejecting' the external backup drive.



After initially suspecting it was something to do with firewire or ACPI (shudder) looking at the stack trace, and the coincidence with removing the drive, it seemed in fact to be an issue in the SCSI subsystem somewhere.  In fact I then worked out the following commands would always repeat the problem:

```bash
umount /mnt/data
scsi_stop /dev/sdh
echo fw1.0 > /sys/bus/firewire/drivers/sbp2/unbind
# The crash happens after next command
echo fw1.1 > /sys/bus/firewire/drivers/sbp2/unbind
```

At this stage I stumbled over an almost identical stack trace inthe [lkml.org](lkml.org) mailing list, which luckily short-circuited my experimentation - learning about the /sys and scsi device manipulation is kind of useful maybe but I had a lot of other things to do as well.

The patch for the problem is documented at [https://lkml.org/lkml/2012/2/8/246](https://lkml.org/lkml/2012/2/8/246).

The next stage, was how to apply it to my system? This is described in the next blog post.

**TD;DR**

The offending stack trace:


    
    
    Mar 16 22:44:47 atlantis3 kernel: [ 2020.140704] sd 15:0:0:0: [sdh] Stopping disk
    Mar 16 22:44:48 atlantis3 kernel: [ 2021.495923] firewire_sbp2: released fw1.0, target 15:0:0
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849206] ------------[ cut here ]------------
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849232] WARNING: at /build/buildd-linux-2.6_3.2.4-1~bpo60+1-amd64-Ns0wYl/linux-2.6-3.2.4/debian/build/source_amd64_none/fs/sysfs/inode.
    c:323 sysfs_hash_and_remove+0x30/0x8b()
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849242] Hardware name: To be filled by O.E.M.
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849248] sysfs: can not remove 'bsg', no directory
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849253] Modules linked in: nls_utf8 nls_cp437 vfat fat rfcomm bridge stp bnep speedstep_lib cpufreq_powersave cpufreq_userspace powerno
    w_k8 ppdev cpufreq_stats lp mperf cpufreq_conservative nfsd lockd nfs_acl auth_rpcgss sunrpc kvm_amd kvm binfmt_misc ext3 jbd fuse ext2 it87 hwmon_vid loop btusb joydev bluetoo
    th rfkill usbhid hid snd_usb_audio snd_usbmidi_lib cx22702 cx88_dvb cx88_vp3054_i2c videobuf_dvb dvb_core rc_winfast tuner_simple tuner_types tda9887 ir_lirc_codec lirc_dev ir_
    mce_kbd_decoder tda8290 snd_hda_codec_realtek firewire_sbp2 ir_sony_decoder snd_hda_intel snd_hda_codec ir_jvc_decoder ir_rc6_decoder snd_hwdep tuner ir_rc5_decoder cx8800 cx88
    _alsa ir_nec_decoder snd_pcm_oss snd_mixer_oss cx8802 cx88xx rc_core i2c_algo_bit tveeprom snd_pcm gspca_ov519 gspca_main v4l2_common snd_seq_midi videodev snd_rawmidi snd_seq_
    midi_event media snd_seq usblp v4l2_compat_ioctl32 videobuf_dma_sg snd_timer snd_seq_device videobuf_core btcx_risc sp5100_tco k10temp edac_core parpo
    Mar 16 22:44:51 atlantis3 kernel: rt_pc parport snd i2c_piix4 tpm_tis tpm edac_mce_amd i2c_core tpm_bios soundcore processor evdev pcspkr thermal_sys mxm_wmi wmi snd_page_alloc
     button ext4 mbcache jbd2 crc16 dm_mod nbd btrfs zlib_deflate crc32c libcrc32c usb_storage uas sg sr_mod cdrom sd_mod crc_t10dif ata_generic ohci_hcd ehci_hcd firewire_ohci fir
    ewire_core crc_itu_t pata_jmicron ahci libahci libata xhci_hcd r8169 mii scsi_mod usbcore usb_common [last unloaded: scsi_wait_scan]
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849487] Pid: 9293, comm: bash Not tainted 3.2.0-0.bpo.1-amd64 #1
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849493] Call Trace:
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849507]  [] ? warn_slowpath_common+0x78/0x8c
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849517]  [] ? warn_slowpath_fmt+0x45/0x4a
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849527]  [] ? sysfs_hash_and_remove+0x30/0x8b
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849538]  [] ? kobject_get+0x12/0x17
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849547]  [] ? mutex_lock+0xd/0x2c
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849555]  [] ? bsg_unregister_queue+0x3f/0x78
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849587]  [] ? __scsi_remove_device+0x34/0xb7 [scsi_mod]
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849613]  [] ? scsi_remove_device+0x20/0x2b [scsi_mod]
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849627]  [] ? sbp2_remove+0x77/0x138 [firewire_sbp2]
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849639]  [] ? __device_release_driver+0x7f/0xca
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849648]  [] ? device_release_driver+0x1d/0x28
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849665]  [] ? driver_unbind+0x56/0x8b
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849674]  [] ? sysfs_write_file+0xe0/0x11c
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849682]  [] ? vfs_write+0xa4/0xff
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849690]  [] ? sys_write+0x45/0x6e
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849699]  [] ? system_call_fastpath+0x16/0x1b
    Mar 16 22:44:51 atlantis3 kernel: [ 2023.849706] ---[ end trace d8b356d84e0828d4 ]---
    Mar 16 22:44:51 atlantis3 kernel: [ 2024.653272] firewire_sbp2: released fw1.1, target 16:0:0
    Mar 16 22:46:06 atlantis3 kerneloops: Submitted 1 kernel oopses to www.kerneloops.org
    
