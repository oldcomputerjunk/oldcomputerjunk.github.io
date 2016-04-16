---
author: admin1
comments: false
date: 2014-06-08 09:00:06+00:00
layout: post
slug: avoiding-your-own-goto-fail-in-a-pinch
title: Avoiding your own Goto Fail in a pinch
wordpress_id: 522
categories:
- infosec
tags:
- code
- security
- software
---

(Note - this article particularly applies to C, C++ and Java like languages.)

Recently I came across some code that had the same kind of latent defect as was responsible for the now (in)famous 'Goto Fail' event [1]. So it was an excuse to write a blog article, I have been a bit <del>slack</del> busy lately.

OK. So ideally your project has decent revision control, with source code guidelines, code review and tools to make it easier to produce high quality code. This should make it easier to reduce the odds of your very own 'Goto Fail' event, right?

Well, here is what often happens in the real world:



	
  * there will be times the project is "experimental", or done in a hurry and not intended to last (and just how many good ideas started off like that, indeed?)

	
  * very often, you have to maintain a legacy codebase

	
  * you need to "ship something yesterday" and haven't been provided with budget to keep it properly maintained; sometimes the accountants win...


With Goto Fail in particular, part of the problem was not really the use of the much maligned `goto` statement in C, but rather lack of adherence to coding standards. In particular, a _**lack of braces** _that limit scope!

Many observers of course jumped on the 'never use goto' meme [2], but one thing I have come to learn in life, almost nothing is black and white. The use of `goto` is like any tool, in this case, one reserved for special situations. To see valid scenarios where `goto` has both readability and performance benefits take a look at the Linux kernel [3]; a well placed goto can remove a tangle of nested if/else/etc as well as provide a key performance improvement, when used properly.

Noting that the Linux kernel is not a normal application; the need for goto will still be extremely unlikely in most mundane applications.


## Summary


Here is a quick and dirty method for auditing a code-base for possible "goto fail"-type defects.


### Requirement


We are looking for the following code fragments:
```C
if (some_condition)
    do_something();

// ...

if (some_condition) do something;
```
These are bad because inadvertent edits can produce:
```C
if (some_condition)
    do_something();
    oops_this_executes_always_instead_of_conditionally();

// ...

if (some_condition) do_something(); oops_this_executes_always();
```
This should be a big no-no!

Instead, we treat either of the following as OK:
```C
if (some_condition) { do_something(); this_executes_conditionally(); }

// ...

if (some_condition) {
    do_something();
    this_executes_conditionally();
}
```

This is actually tighter than the Google Style Guide [4]; consider it wise to plan for multiple developers editing future code, some who may be inexperienced or "just passing through" - on the theory of if they expend the least energy, if the braces are there, they will use them!


### Method #1 - from a Linux / Unix shell


```bash
find -name "*.c*" -exec egrep -nH '^[\t ]*if[\t ]*\([^{]*$' {} \;
```

Extend the `-name` clause as required.

This works by finding all lines where the first non-whitespace is the `if` statement, and then reports if there is NO opening brace anywhere after on the same line.  It handles any whitespace combination.


### Method #2 - the following regular expression may work in Microsoft Visual Studio


_Caveat: I haven't had a chance to test this ye, so if it is wrong, please let me know!_

```plain
if ([^{]*$
```


### Notes


Now, these wont catch everything: if anything 'creative' is happening inside a preprocessor macro for example; also this may trigger false positives if you have a string constant that includes the text `" if ("`.  It is also probably not resilient against nested statements on the same line, if your guidelines allow it (e.g. ` if (blah) { for (;;)` would not be caught without modifying the expression.  

The above expressions, as-is, will also produce many false positives, if your coding guideline allows the following:

```C
if (some_condition)
{
    do_something();
    this_executes_conditionally();
}
```

I'll leave it as an exercise for the reader to adjust the regular expression to suit... it should extend to more complex situations if required.  You could also use `sed` or `pcregrep` or even `perl -e`for example.

if you have to remediate a very large code base, you can redirect the output to a file and measure progression over time. You could script out any false positives as well.

[1] [http://nakedsecurity.sophos.com/2014/02/24/anatomy-of-a-goto-fail-apples-ssl-bug-explained-plus-an-unofficial-patch/](http://nakedsecurity.sophos.com/2014/02/24/anatomy-of-a-goto-fail-apples-ssl-bug-explained-plus-an-unofficial-patch/)

[2] [http://en.wikipedia.org/wiki/Considered_Harmful](http://en.wikipedia.org/wiki/Considered_Harmful)

[3] [https://web.archive.org/web/20130410044210/http://kerneltrap.org/node/553/2131](https://web.archive.org/web/20130410044210/http://kerneltrap.org/node/553/2131)

[4] [http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml#Conditionals](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml#Conditionals)
