---
author: admin1
comments: false
date: 2015-04-23 12:24:48+00:00
layout: post
slug: why-is-the-arduino-ide-so-stupid
title: Why is the Arduino IDE so stupid?
wordpress_id: 658
categories:
- tech
tags:
- arduino
- embedded
---

If I perform the following actions:



	
  * File, New
Opens a new editor window. Reasonable enough, although I would have preferred a default single-window GUI model like QtCreator or even gedit.

	
  * File, Save
Opens a save as dialog. In spite of the Arduino 'sketchbook' directory, it opens in my home directory.

	
  * New Folder
Creates a directory New Folder, but doesn't shift the focus to it, leaving you confused when this is done in a directory with a lot of files...

	
  * Click on 'New Folder' and rename it, say, Test123

	
  * Navigate into Test123/

	
  * Type in a filename for the project, say, TestTest1

	
  * Hit save.
So now Arduino IDE dutifully ignores what I typed and proceeds to create a tab called 'Test123'.
It will even do this if 'Test123/' already existed.
What?

	
  * File, Save As.
It forgets where you where in the hierarchy and starts in the home directory again(!)

	
  * Navigate to Test123/ intending to use it as a container for multiple projects

	
  * Type in a filename, say Hello, then hit Save

	
  * The sketch is _still_ called Test123.
What?


So insanely enough, it seems you essentially create a director and _thats_ where the sketch gets its name.  Within that directory it creates a file with the same name with the extension '.ino'

Lets try something else:



	
  * From the shell, create a directory, Test456 and create a readme.txt file, and a directory Test456a and a file Test456a/readme2.txt

	
  * File New

	
  * File save

	
  * Navigate into Test456

	
  * Type in helloworld for the name

	
  * Again, the project gets called Test456

	
  * But take a look in the directory Test456: the contents are now gone (all, including the sub directory Test456a) and replaced with Test123.ino
**Wait, what?** **THIS SHOULD NEVER EVER HAPPEN!!!!**


Luckily I discovered this in a directory in a git working copy with no modifications so I didn't lose anything important.

Testing done using Arduino1.5.8 amd64 for Digispark. So its a little out of date but not exactly the oldest either.

I have used Arduino before and to be honest I don't recall it being this stupid, but maybe I just got lucky.

One difference is this time I got sick of the massive latency opening the windows and tried a few different Java JRE (openjdk6, openjdk7, gcc4.7-jre) before discovering that with gcc4.7-jre the menus are as snappy as the openbox right click menu, or even a (*sharp intake of breath*) brand new Windows 7 corporate desktop... maybe there is some API implementation difference between the JRE's that affects the file save dialog functionality.

I don't seem to have any issues opening projects.

So my workflow for creating a new project now consists of:

	
  * From the shell, create a directory in the relevant part of the git working copy I am using

	
  * Create a new empty .ino text file with the same name as the directory (or copy a template I made)

	
  * Open it with the IDE and start working



