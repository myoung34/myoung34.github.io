---
title: Complete Continuous Deployment Using Puppet and Pulp - Overview
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/continuous-deployment-using-puppet-and-pulp-overview
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - Linux
  - puppet
tags:
  - linux
  - puppet
  - pulp
  - jenkins
  - git
---
This will be by far my biggest post to date. I've been asked from multiple sources about the system I've set up at my current company, so this is a very small version of it. By the end of this, assuming you follow along, you'll have a multiple node system set up that will deploy a linux service, packaged as an RPM, to internal Yum feeds, with Puppet along the way to keep this continuously deployed. The puppet module I'll write to do this will also be continuous, with all changes to it causing tests to run, and r10k to redeploy them if successful.<!--more-->

This series will be in this layout:

  * Overview (this page)
  * [Part 1 - The Nodes and their purpose in this stack](/post/continuous-deployment-using-puppet-and-pulp-pt-1-the-machines/)
  * [Part 2 - Creating a small linux service that causes new RPM builds when changes are seen via Jenkins](/post/continuous-deployment-using-puppet-and-pulp-pt-2-the-app/)
  * [Part 3 - Creating a puppet manifest to deploy and maintain the previous package](/post/continuous-deployment-using-puppet-and-pulp-pt-3-enter-puppet/)
  * [Part 4 - Managing the puppet manifests so that changes also run tests and re-deploy to the ENC (foreman)](/post/continuous-deployment-using-puppet-and-pulp-pt-4-puppet-jenkins/)
  <!--* [Part 5 - Simplifying your life with vagrant to develop and maintain this setup without worrying about nodes that are subscribed to the packages/puppet manifests]() - not available yet-->

#####Note
This is by NO MEANS a copy/pasta guide. I'm going based on the assumption you just want to see the high level, some code, and the flow. I'll link to guides on how to set up the nodes such as the Pulp server, Foreman server, etc by I won't go into details that don't tie into the end-goal.

###Workflows

  1. Developer workflow
    * Continuous integration of puppet modules
      * Make a change to a puppet module
      * Jenkins deetects the change and runs tests
      * Jenkins tells the ENC (puppet master) to re-deploy its modules
    * Continious Deployment of an application
      * Make a change to your git repository (the package code base)
      * Jenkins detects the change and runs tests
      * Jenkins builds an RPM
      * Jenkins uploads that RPM to a Pulp repository
        * These can be 'feeds', such as a change to *dev* branch creating an RPM in an *unstable* repository. A change to *master* branch would create an RPM in a *stable* repository*
  2. Server workflow (puppet agents)
    * Servers subscribe to a repository (stable or unstable)
    * A Cron job continuously checks for changes to the package in this Yum feed
    * A new package is detected and is bootstrapped to the server

###Glossary
  * Pulp - a Yum repository client/server similar to Spacewalk, but much more light weight. Think aptly for Redhat
  * Jenkins - A continuous integration server written in java
  * Puppet - A client/server configuration  management suite
  * Foreman - An ENC for puppet. Allows you to do things such as reports, monitoring etc for Puppet nodes, similar to Puppet Dashboard
  * Vagrant - A suite that allows you to bootstrap virtual machines, hiding the implementation. Think a CLI that lets you build a base VM, share it, and swap it out for different providers such as vmware, virtualbox, AWS, etc
