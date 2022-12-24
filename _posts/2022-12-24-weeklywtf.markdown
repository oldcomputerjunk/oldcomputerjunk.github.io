---
layout: post
slug: weekly-wtf-22-51
title:  "The stupidest smallest things are sometimes the hardest to work out how to fix"
date:   2022-12-24 00:00:00
categories:
- howto
tags:
- hacking
- utf8
---

# WTBOMF (or why is plain text copy paste broken on the web)

Ever copy/pasted a shell script code fragment from a web page into vi/vim or another text editor and then had inexplicable errors running the end result? What about C code not compiling? And then if you `cat` the file you can't for the life of you see what the problem is?

When this happens, I'm getting quicker at running the affected file through `xxd` and discovering unexpected binary bytes in my file, these are in fact invisible UTF-8 characters. I mean, I know there is a whole pile of fun to be had with these incorrigible byte order marks and what not, I've sat through various talks at conferences that I realise they do have a valuable accessibility benefit, and they let us have those all important emoji, but seriously, when going the other way 99.999% of the time anyone copying selected text from a web browser wants the actual text, not some invisible scrambled egg of binary soup!

Anyway, if you discovered you pasted this stuff you can fix it few ways

(I'll use vi and vim interchangeably below although it is highly likely that some stuff only works in vim not the minimal vi)

## Discovery

Try running your script through either `xxd` or `od` on Linux / MacOS (or wsl)

Sample output from `od -Ax -tx1z < somefile.txt` 
```
000000 2e 67 6c 6f 62 61 6c 20 5f 52 65 73 65 74 0a 5f  >.global _Reset._<
000010 52 65 73 65 74 3a 0a c2 a0 4c 44 52 20 73 70 2c  >Reset:...LDR sp,<
000020 20 3d 73 74 61 63 6b 5f 74 6f 70 0a c2 a0 42 4c  > =stack_top...BL<
000030 20 63 5f 65 6e 74 72 79 0a c2 a0 42 20 2e 0a     > c_entry...B ..<```
```

Via `xxd somefile.txt` instead:
```
00000000: 2e67 6c6f 6261 6c20 5f52 6573 6574 0a5f  .global _Reset._
00000010: 5265 7365 743a 0ac2 a04c 4452 2073 702c  Reset:...LDR sp,
00000020: 203d 7374 6163 6b5f 746f 700a c2a0 424c   =stack_top...BL
00000030: 2063 5f65 6e74 7279 0ac2 a042 202e 0a     c_entry...B ..
```

Note after each line end (`0x0a`) there is the two hex bytes `0xc2` and `0xa0`

Technically, this is UTF-8 for a 'non breaking space', same as HTML entity `&nbsp;` you can read about it at https://www.fileformat.info/info/unicode/char/00a0/index.htm

WHen the original text was pasted into vim it doesnt show anything for these bytes by default, even though vim usually shows other control characters like CTRL+X as `^X`

The resulting file doesn't show anything when output via `cat` either.

(Somewhat related, also watch out for things getting even more confusing if using vim in Windows Terminal over ssh or WSL instead of native Linux or Mac, where presses like CTRL+V dont work as they are used for the clipboard instead. Easily fixed eventually - right click on the Windows Terminal window bar (not on a tab), go to Settings, Actions, find **paste (Ctrl+V)** and delete it, thi still leaves CTRL+SHIFT+V and CTRL+INS to do the same thing)

## Method 1 - sed

We can run sed over it and with some bash trickery get it to replace the dodgy bytes:
```
sed -i -e 's/'$(echo -e '\xc2\xa0')'/  /g' somefile.txt
```

Here, `-i` means in place edit the input file, `somefile.txt`, and our search expression is build by using the ability of Linux/bash echo to output arbitrary binary bytes.

Note we replace them with two white spaces (or maybe something else, depending on what you copied) or the end result will not have the expected indentation.

## Method 2 - show the things in vim itself

Change the current encoding from the default (likely utf8) to Latin 1:
press `ESC` then `:set encoding=latin1`

And like magic:

![Vim in Windows terminal showing diamonds for unprintable UTF characters](/images/magical-diamonds-1.png){:height="100%" width="100%"}

Now you can manually delete them, or maybe even find a way to use search and replace. 

# Why does this happen

Lots of blog rendering frameworks include modes for syntax highlighted formatting with line numbers and the like. Some of these simply apply CSS over a `<pre>` block for example.

Others get a bit fancier however.

If you find an example, it can be illustative to hit CTRL+SHIFT+I in your web browser and take a look at what you just copied. It might look like this, full of `DIV` and `TD` element:
```
<table cellspacing="0" cellpadding="0" border="0"><tbody><tr><td class="gutter"><div class="line number1 index0 alt2">1</div><div class="line number2 index1 alt1">2</div><div class="line number3 index2 alt2">3</div><div class="line number4 index3 alt1">4</div><div class="line number5 index4 alt2">5</div></td>
<td class="code"><div class="container"><div class="line number1 index0 alt2"><code class="plain plain">.global _Reset</code></div><div class="line number2 index1 alt1"><code class="plain plain">_Reset:</code></div>
<div class="line number3 index2 alt2"><code class="plain spaces">&nbsp;</code><code class="plain plain">LDR sp, =stack_top</code></div>
<div class="line number4 index3 alt1"><code class="plain spaces">&nbsp;</code><code class="plain plain">BL c_entry</code></div>
<div class="line number5 index4 alt2"><code class="plain spaces">&nbsp;</code><code class="plain plain">B .</code></div></div></td></tr></tbody></table>
```

Look, there are `&nbsp;` in that lot! And the browser helpfully decides to keep then when copying or pasting.

If you are on your toes, sometimes these syntax highlighters have a 'copy' button which actually grabs the expected text. But sometimes, you just want to grab what you selected...

Note, I'm not blaming content authors for this, they like me just trying to share information. Possibly even this very blog does it somewhere, especially the older articles which got imported to Jekyll from Wordpress. I should probably go and check, hey!  But it does say something about the state of our tooling, everything seems to find a way to disrupt the end user experience in one way or another as a trade off of some kind...

## While we are here

FWIW you can also turn on a mode in vim that shows tab and space characters (but not invisible UTF) which can be useful in Makefiles and the like.
Try using: `ESC` then `:set list`, this also has an argument that lets you customise what gets shown.

# Lastly, don't get pwned

Check out this video from Defcon 30 earlier this year, showing how Emoji can be used for shell code... https://www.youtube.com/watch?v=E8puAkalMRQ

And then there are the hidden paste attacks: http://lifepluslinux.blogspot.com/2017/01/look-before-you-paste-from-website-to.html

Be careful out there! 👾 👽 💩
