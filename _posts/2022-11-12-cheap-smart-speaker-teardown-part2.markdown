---
layout: post
slug: cheap-smart-speaker-teardown-part2
title:  "Cheap Smart Speaker Teardown part 2"
date:   2022-11-12 02:00:00
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

This is Part 2 in a series. [Part 1](2022-11-12-cheap-smart-speaker-teardown-part1.markdown) is a photo essay of the hardware teardown.

# Recap

To recap, the objective is to complete a physical tear down (done), perofrm a black box reconaissance, dumping the firmware and make a preliminary attack surface analysis. With luck, then put it back together retaining access to the running internals.

# Part 2 - Passive Reconaissance

I mean, usually you might do this first, but really, I would usually do it in parallel because you find interesting hints while pulling things apart. In this case I stayed patient and captured a complete set of photos before I started.