---
author: admin1
comments: false
date: 2015-04-27 11:55:13+00:00
layout: post
slug: using-an-i2c-rtc-with-the-carambola2-or-any-openwrt-modified-router
title: Using an i2c RTC with the Carambola2 (or any OpenWRT modified router)
wordpress_id: 660
categories:
- linux
tags:
- embedded
- linux
- openwrt
---

Using i2c with a modded router is simple enough, if you have two spare GPIO then the module package kmod-i2c-gpio-custom allows selected GPIO pins to be bound to SCL and SDA respectively when the module is loaded.

However for inexplicable reasons the ability to bind an i2c RTC module to the Linux hardware clock is disabled by default by the OpenWRT configuration mechanism for ar71xx and other consumer router architectures, and there is no way to turn it on without patching!

Regardless, here is how to use an i2c RTC with the Carambola2 or any other ar71xx architecture router (e.g. WRTnode, etc.)



	
  * Patch the file **target/linux/ar71xx/generic/target.mk** as follows:
[crayon lang="plain"]
-FEATURES += squashfs
+FEATURES += squashfs +rtc
[/crayon]

	
  * Patch the kernel configuration **target/linux/ar71xx/config-3.xx** where (**xx** depends on your version of OpenWRT) as follows:
[crayon lang="plain"]
+CONFIG_RTC_CLASS=y
+CONFIG_RTC_DRV_DS1307=m
[/crayon]

	
  * Note: the kernel configuration can be modified via the kernel build system using the command **make kernel_menuconfig**

	
  * Note: add other kernel i2c RTC modules as required

	
  * Add the module to your image:
[crayon lang="plain"]
CONFIG_RTC_SUPPORT=y
CONFIG_DEFAULT_RTC_SUPPORT=y
CONFIG_PACKAGE_kmod-rtc-ds1307=y
[/crayon]

	
  * If you have previously built OpenWRT then remove the **tmp/** directory, or the change '+rtc' will be ignored and the DS1307 module will not be included in your image

	
  * Run **make defconfig**

	
  * Build your image: **make -j2**


If everything worked, then the the file **/lib/modules/3.xx.../rtc-ds1307.ko** should be in the resulting image

Following is an aggregation of information I was already able to find elsewhere on the net.





  * Ensure that **i2c-tools** package is installed as well. This may require the 'oldpackages' feed.


  * Configure the module as follows by creating a file **/etc/modules.d/99gpio-i2c-rtc**


  * You can also put this file into **files/etc/modules.d/99gpio-i2c-rtc** for it to be automatically added to your image


  * Create the following content, where in this example 18 == SDA pin id and 19 == SCL pin id
[crayon lang="plain"]
i2c-gpio-custom bus0=0,18,19
rtc-ds1307
[/crayon]



  * There are additional arguments controlling delays, etc.; refer to **package/kernel/i2c-gpio-custom/src/i2c-gpio-custom.c**


  * Create a script, **/etc/init.d/rtc-driver** to load the device driver and set the time.
[crayon lang="plain"]
#!/bin/sh /etc/rc.common
logger "Setup i2c RTC"
echo ds1307 0x68 > '/sys/class/i2c-dev/i2c-0/device/new_device'
if hwclock | grep 'Jan' | grep -q 2000 ; then
  logger "RTC appears to have a flat battery..."
else
  logger "RTC set hwclock"
  hwclock -s
fi
[/crayon]



  * Create a symlink...
[crayon lang="plain"]
ln -s /etc/init.d/rtc-driver /etc/rc.d/S11rtc-driver
[/crayon]



  * Note, if you are running ntp that will take over anyway, but for system with an intermittent or no network connection, or if the network is down on boot, the RTC will provide a better time than 1 Jan 2012 or whatever...


You can test the above out before scripting it by booting the system and manually stepping through:
[crayon lang="plain"]
modprobe i2c-gpio-custom bus0=0,18,19
i2cdetect -l
modprobe rtc-ds1307
echo ds1307 0x68 > '/sys/class/i2c-dev/i2c-0/device/new_device'
hwclock
[/crayon]

Enjoy!

PS Dont forget pullup presistors, and take care interfacing between 5V and 3.3V systems and peripherals...
