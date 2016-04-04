---
author: admin1
comments: false
date: 2014-01-06 13:19:36+00:00
layout: post
slug: on-linux-conf-au-and-presentation-logistics
title: On Linux.conf.au and Presentation logistics
wordpress_id: 466
categories:
- tech
tags:
- impress
- LCA2014
---

## How to diff LibreOffice Impress presentations in git


These days I keep track of presentation material using git (like many things). One handy trick is you can ``git diff`` an ODP file and have the text changes show like any other code.

The following procedure will enable this for Debian, YMMV.



	
  1. `sudo apt-get install odt2txt`

	
  2. Edit the file `.gitattributes` in the repository containing the ODP files. Note that this file is interpreted at each level in the directory structure

	
  3. Add the following line to the file:
`*.odt diff=odt`

	
  4. You should probably commit .gitattributes as well.


I haven't yet worked out a way to make this global using 'git config' which is a little annoying.

Another quick tip,  to redock the 'slides' pane if you manage to set it "loose", hold CTRL and double click the word 'slides' at the top _inside_ the pane window.

And lastly, if anyone can tell me how to get Impress Presenter Mode to actually work with xrandr without obscuring the actual presentation that would be awesome...

Managing to avoid making last minute tweaks to a presentation is another problem not solvable with code ;-) The problem is I always think of improvements each time I re-read my talk, and also once the conference has started I always think of good techniques to try after watching the other speakers...



(One miniconf talk down, one to go!)
