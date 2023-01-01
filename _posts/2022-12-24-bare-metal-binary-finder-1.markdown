---
layout: post
slug: bare-metal-binary-finder-1
title:  "Prospecting for bar metal libc functions"
date:   2023-01-31 00:00:00
categories:
- howto
tags:
- hacking
- embedded
- reversing
---

# Introduction

When reversing bare metal embedded binaries in particular, but also other executables, finding statically linked C library functions is a chore. The obivous but tedious method involves:
- start with strings
- find places where they are used as arguments, to functions that are quite short
- decompile those functions and see if they look roughly like what `strncmp`, `strcat`, `strcpy`, `vsnprintf` do, etc.
- hopefully these are called from multiple places
- this feels a bit like solving a crossword - sometimes you think you have a right answer, then after you reverse some other calling function it actually turns out to be something else

Now this kind of thing lends itself to getting the computer to do the work, one would think. Not just find known standard C functions - any set of bound for known input function one might consider.

It turns out, there are some tools (and likely others I haven't found, but then I wouldn't have a reason to write these articles)

- Ghidra has a Function ID database mechanism. This is well suited to finding known functions from public widely used shared libraries, as it seems to be hash based. But in the embedded world, it is much more likely bits of the library got recompiled as part of the build and could have subtle optimised differences, so the hash won't match

- This plugin set for Ghidra [https://github.com/tacnetsol/ghidra_scripts]() includes a function called 'leafblower` that a hueristic to find 'leaf' functions - functions that dont call other functions - and list those that meet some criteria - called a few times, has 2 or 3 arguments, etc. and attempt to guess what it might be. But you still need to confirm what they all do...

- Presumably there are machine learning based tools that do this. They will ultimately have some probability of match, it would be good to know for sure. Also, it depends on what they have been trained on.

- And doubtless some people (if not already) will paste code into ChatGPT, and then automate that - again, the results could be incorrect, more to the point, the VC's will be looking for their pound of flesh for using this before too long, and in any case, if I have this 8-core computer with 60000x the processing power that took Apollo to the moon sitting on my lap I don't see why I should ask and wait for requests from someone else computer to get this task done (you that is what the cloud is, right?)

Anyway, I figured, given I've spent much of my career building software intensive systems, maybe we could combine a simple unit test framework with an emulator and push binary snippets from Ghidra or Radare2 through that?

The hypothesis:
- it should be feasible to construct a set of unit tests for a well defined subset of the standard C library that will uniquely identify what a function does
- having that, it should be possible to devise a framework that then allows additional arbitrary functions to be specified
- the tests can then run a binary function extracted from a dissasembly tool such as Ghidra or Radare2, using an emulated processor sich as via Qemu or perhaps Unicorn engine

Advantages:
- this will be independent of the architecture (ARM, MIPS, x86)
- with the right API it should be able to be connected to Ghidra, Radare2, and be fed functions that have been identified using the initial analysis
- if a unit test is functionally comprehensive in the simplest form possible, even though this is a brute force search it should be both fast and produce correct results

Disadvantages:
- What this approach won't do (at least, immediately)
  - identify inline code
  - how to handle globals & statics - we need to somehow send the entire memory map, and/or use Ghidra pcode to find memory accessess first (and what about those that need setting?)