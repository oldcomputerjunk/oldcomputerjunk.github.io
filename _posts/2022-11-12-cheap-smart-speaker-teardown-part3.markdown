---
layout: post
slug: cheap-smart-speaker-teardown-part3
title:  "Cheap Smart Speaker Teardown part 3"
date:   2022-11-18 02:00:00
categories:
- infosec
tags:
- embedded
- hacking
- reversing
- android
- hardware
- home-automation
---

# Introduction

Last month I picked up a cheap smart speaker from a local electronics retailer and this weekend I was rained in, so took the opportunity to have some fun and practice doing a teardown and firmware dump

This is Part 3 in a series. [Part 1](2022-11-12-cheap-smart-speaker-teardown-part1.markdown) is a photo essay of the hardware teardown and [Part 2](2022-11-12-cheap-smart-speaker-teardown-part2.markdown) describes some passive reconaissance I undertook, and identification of key integrated circuits.

# Recap

To recap, the objective is to complete a physical tear down (done), perform a black box reconaissance (done), dump the firmware and make a preliminary attack surface analysis. With luck, then put it back together retaining access to the running internals.

The main processor I am 95% confident is an ARM 64-bit with 512MB of DDR; there is a bluetooth + wifi SOC with Linux drivers, and a good chance this system runs Linux. There is a DAC, a secondary microcontroller, and a 1GB flash chip. The PCB hints there is a serial port but we haven't yet located it.

# Part 3 - Serial Port


As noted in  [Part 1](2022-11-12-cheap-smart-speaker-teardown-part1.markdown), the upper side of the PCB has labels for UART Rx & Tx. There is no obvious corresponding pins nearby!

![UART Labels](../images/cheap-smart-speaker-teardown-part3/100-uart-labels.png){:height="100%" width="100%"}

I started with that unpopulated connector. WIth the multimeter- it seemed that what I will call Pin2 was ground (it had continuity with the can and some other ground points), and Pin 4 was 3V. The others were inconclusive. Indeed, when I power cycled the device, with the cro on Pin 4 you could see it come up.

I had the photos in hi-res on the big screen, so I had a good search. There are not so many obvious points on the top of the PCB, but the underside was covered with test points.

Here is where I hit a red herring. It is very easy to make mistakes when searching for an unknown; but then, if there was already a pulished schematic I probably wouldn't be needing to reverse engineer anything!

On the underside in the area of the labels, there are a number of pins. There are the pins marked (A), but they had no label. But I got a interested because at (B) are some pins - these are marked TPD5 and TPD6 - and I noted most of the other test points were labelled TPMxx. I wondered - is the 'D' the difference?

![UART Candidate pins](../images/cheap-smart-speaker-teardown-part3/110-uart-maybe.png){:height="100%" width="100%"}

I buzzed all four of these (and several others) with the multi-meter, but as expected, this was inconclusive. Sometimes, you can speculate that a pin might be some kind of communications pin, or active, if the voltage is "in-between". In particular, some of these pins were ground, others at 1.8V. Of course, this device is actually 1.8V for much of its logic, not 3v3 like common Raspberry Pi boards.

Anyway in the end, I soldered some wires to (B) and connected them in turn to the oscilloscope, whilst power cycling the device. My hope was that on boot there would be bursts of what might possibly be serial traffic.

This was inconclusive. I seemed to get a pulse, ground, to high, to low, to high, but with a long duration (perhaps half a second). This did converm the 1.8V logic level though.

Then I shook my head, because on further inspection, I realised the pins at (A) were very much directly underneath the UART labels! It was so obvious in hindsight. I removed the wires and instead soldered them to (A).

This time, with the CRO connected, some joy - what looked like serial!

Excitedly, I connected this to the Tx/Rx lines on my Pi400, and fired up picocom. But no joy - just junk!

```
pi@pi400:~ $ picocom -b 115200 /dev/ttyS0 
picocom v3.1

port is        : /dev/ttyS0
flowcontrol    : none
baudrate is    : 115200
parity is      : none
databits are   : 8
stopbits are   : 1
escape is      : C-a
local echo is  : no
noinit is      : no
noreset is     : no
hangup is      : no
nolock is      : no
send_cmd is    : sz -vv
receive_cmd is : rz -vv -E
imap is        : 
omap is        : 
emap is        : crcrlf,delbs,
logfile is     : none
initstring     : none
exit_after is  : not set
exit is        : no

Type [C-a] [C-h] to see available commands
Terminal ready
�	�      �1�z��щ��	�	1	{����	z)
                                                          α	!��)����	a�1�z�		��	!��
```

Usually this junk indicates the wrong speed. So I cyced through all the common ones - 115200, 57600, 38400, 19200, 9600... just more junk. At this point I took a break and did some more searching. Maybe it wa 230400? Until now, 115200 was the most common (with the exception of the ESP8266, which has a wierd rate at 47300 or something during boot and then switches to 57600)

Back to the CRO, and this time I spent a bit more time. I keep triggering until something filled the screen with a diversity of high and low of various widths, and then I measured the thinnest. The pulses and gaps were approx 1 microsecond... inverse of approximately 1MSPS, or close enough to 921000 baud... so I reconfigured the Pi again, and this time, running the command picocom -b 921000 - eureka!

This is what I had on screen:

```
pi@pi400:~ $ picocom -b 921000 /dev/ttyS0 
picocom v3.1

port is        : /dev/ttyS0
flowcontrol    : none
baudrate is    : 921000
parity is      : none
databits are   : 8
stopbits are   : 1
escape is      : C-a
local echo is  : no
noinit is      : no
noreset is     : no
hangup is      : no
nolock is      : no
send_cmd is    : sz -vv
receive_cmd is : rz -vv -E
imap is        : 
omap is        : 
emap is        : crcrlf,delbs,
logfile is     : none
initstring     : none
exit_after is  : not set
exit is        : no

Type [C-a] [C-h] to see available commands
Terminal ready
[UPG]IDLE
         [UPG]IDLE
                   Battery voltage: 3975 mV
                                           Battery 75% ~100% 
                                                             <KERWIN>led msg is 58
  [UPG]Power State:BAT_POWER_4
                              [UPG]IDLE
                                       [UPG]IDLE
                                                [UPG]Player State:PL_STOP
                                                                          Battery voltage: 3979 mV
                  Battery 75% ~100% 
                                    <KERWIN>led msg is 58
                                                         [UPG]Power State:BAT_POWER_4
     [UPG]IDLE
```
