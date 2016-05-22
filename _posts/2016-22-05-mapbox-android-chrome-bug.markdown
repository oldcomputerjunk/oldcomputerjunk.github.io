---
layout: post
slug: mapbox-android-chrome-bug
title:  "Mapbox on Chrome on Android"
date:   2016-05-22 01:00:00
categories:
- tech
tags:
- javascript html5 mapbox
---

I've been working on a web app for a friend that includes geolocation functionality.
I have been prototyping using the [mapbox-gl.js](https://www.mapbox.com/mapbox-gl-js/api) API, and combined with the [turf.js](http://turfjs.org) library and have managed to create a very functional client side application in good time.
I have been very impressed with how easy it is to create a map and overlay a variety of featues.

Along the way I've had a refresher in Javascript, having come up to speed with HTML5 features and Promises, require.js and unit testing using [qunit](https://qunitjs.com). I am finding development very productive and much less frustrating with the diamond of death out of the way...

And I love the newer functionality like HTML5 [SpeechSynthesis](https://developer.mozilla.org/en-US/docs/Web/API/SpeechSynthesis)!

### Multiple Browser Fun

Of course, the browser merry-go-round still exists even if it has been held at bay for a while.

I have struck a bug where the webGL part of mapbox seems to have some issues when run in Chrome on my Android tablet. Which is a bit annoying because the tablet is one of my main targets for this app.

Essentially, it seems that once you try and interact with the map, such as adding features, or using two map instances on one page, the map will "blank out". Even doing nothing other than showing the map, and then attempting to finger drag causes the problem. This seems to be specific to the latest version of Chrome on Android; this demo works everywhere else I tried: Firefox (Linux, Windows, Android), Chrome (Linux, Windows), Safari (iPad) even IE (Windows)

There is a [demo here](https://pastcompute.github.io/html5-demos/).

Demo Working (on Firefox on Android)

<img src="/public/2016-05-22_19.54.30.png" alt="Not Working on Chrome" class="inline"/>

Demo Not Working (on Chrome on Android)

<img src="/public/2016-05-22_19.54.21.png" alt="Working on Firefox" class="inline"/>

This is happening on my Acer A1-810 running Android 4.2.2 and Chrome 52.0.2739.3 which appears to be the latest version.

One thing I noticed is a bunch of warnings in `adb`. Possibly this provides some clue.

```
[INFO:CONSOLE(0)] "[.Offscreen-For-WebGL-0x5f0de500]RENDER WARNING: there is no texture bound to the unit 1", source: http://X.X.X.X/~X/bugdemo.html
```

I'll post an update if I make any further progress.

