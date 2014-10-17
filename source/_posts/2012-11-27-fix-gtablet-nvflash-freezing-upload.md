---
title: (FIX) GTablet NVFlash Freezing During Upload
author: myoung
excerpt: "If for some reason you're trying to NVFlash your gtablet (probably with gtab.nvflash.1.2.branch.20110508 with cwm5504 or gtab.nvflash.1.2.branch.20110508 with cwmv3028.zip), and it's freezing on the flash, here's what I did."
layout: post
comments: true
permalink: /post/fix-gtablet-nvflash-freezing-upload/
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - Android
  - Fix
  - Performance
tags:
  - android
  - beasty clemsyn
  - bug
  - cwm
  - fix
  - froyo
  - gtablet
  - nvflash
---
I currently own a ViewSonic GTablet, which I do love the few times I use it, but if you&#8217;re like me, you <del datetime="2012-11-27T04:17:03+00:00">tamper</del> break things you own.<!--more-->

My GTablet runs [][1] which I prefer since it&#8217;s freaking fast, however with the performance, there are a few bugs, all of which I seemed to hit in a single move&#8230;.so this is a &#8220;Don&#8217;t do&#8221;:

*   Putting it in airplane mode goes ape-shit. There was no way to get it out, including reboots and settings changes.
*   To fix this, I hit &#8216;Restore to factory defaults&#8217;&#8230; **Don&#8217;t ever do this!**
*   This fubared everything, which means restore old Rom&#8230;yay for my free time tonight.

So far, whatever&#8230;easy fix right?  
Well if you&#8217;ve made it to this page, you see the title, and that&#8217;s what I&#8217;m here to fix.  
If for some reason you&#8217;re trying to NVFlash your gtablet (probably with gtab.nvflash.1.2.branch.20110508 with cwm5504 or gtab.nvflash.1.2.branch.20110508 with cwmv3028.zip), and it&#8217;s freezing on the flash, here&#8217;s what I did.

Ready?  
In the *gtablet.cfg* file, swap the partition in question (the one freezing) with the previous one, but swap it on the filesystem level.  
For example, my Partition 7 froze on upload, so the diff between the original and my fix: <a href="https://gist.github.com/4152380" title="https://gist.github.com/4152380" target="_blank">gtablet.cfg-diff</a>  
If you want my full fixed file (assuming yours broke on partition 7 as well): <a href="https://gist.github.com/4152387" title="https://gist.github.com/4152387" target="_blank">gtablet.cfg/4152387</a>

 [1]: http://forum.xda-developers.com/showthread.php?t=1084573 "Beasty Clemsysn (Froyo)"
