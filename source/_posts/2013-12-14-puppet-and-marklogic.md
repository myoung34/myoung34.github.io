---
title: Puppet and MarkLogic
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/puppet-and-marklogic
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - deployment
  - Linux
  - puppet
  - Uncategorized
  - windows
  - marklogic
tags:
  - deployment
  - git
  - linux
  - puppet
  - windows
  - marklogic
---
I've recently been writing a few modules for my current job, and have seen some missing modules on the [puppetforge](https://www.puppetforge.com). One of which is something to maintain a [MarkLogic](https://www.marklogic.com) instance.<!--more-->

Apparently, however, I apparently like to pick inherently complicated work. MarkLogic has an interesting EULA. They have RPMs to install their software, but the EULA prevents redistribution. That means there is no way to have a central YUM repository. Also in order to download the .rpm file off of their website, you have to log in. That means you can't just curl the latest package. In terms of a module, this kind of sucks because it means that you can't just install the module and have ``ensure => latest`` since there's no way of keeping an internal package store. 

How does this suck for me? I like to test. It's my new thing. In order to test your module, you need consistency, and for this that's hard to do without violating their terms. My solution? I have a super secret vagrant .box file that has a local Yum repository with the versions I know it supports (everything over version 6). No big deal, sort of.

Did I mention the software also requires activation? That affects testing too, because I can't just hardcode my license information even though it's free and easily obtainable. My solution was to force the user to have a ``~/.marklogic.yml`` file that contains the information. None of this means anything to the normal use who wishes to install marklogic, they just need a Yum repository set up that holds the MarkLogic version. It does mean something to me in terms of making something that's easy to fork and fix if there are issues.

TL;DR: [My module is here.](https://www.puppetforge.com/myoung34/marklogic)
