---
title: What I Learned From PuppetConf 2013
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/puppetconf2013
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
tags:
  - deployment
  - git
  - linux
  - puppet
  - windows
---
Wow. Work has been crazy. Enough for me to have no activity here. Time to change that, but putting in a blog post that might resemble a cracked article.<!--more-->

I recently attended Puppetconf 2013 in San Francisco, and it was amazing (except the food). I love that city now. Regardless, I went into this conference having written a few very incorrect modules that did work, although I'm unsure how in hindsight. I decided to finally put in a post some of the takeaways I gathered. Ironically, almost all of my best information came from sitting down on developer day, which was a topic themed sit-down to talk to like minded people and puppet developers. The idea being to get them to look at your code, help you out, or just answer questions. This. Was. Awesome. Much thanks to Ashley Penney and Rob Reynolds for dealing with me.

Things To Take Away From My Conference Experience
=================================================

1. Module best practices
    * The [style guide](http://docs.puppetlabs.com/guides/style_guide.html) is your bff.
    * Tests, tests tests. 
        1. [rspec-puppet](http://rspec-puppet.com/tutorial/) lets you write rspec tests for your manifests.
        2. [rspec-system-puppet](https://github.com/puppetlabs/rspec-system-puppet) lets you write system tests for a full stack (catalog)
    * [Puppet-lint](http://puppet-lint.com/). Use it. Love it. Always run with _--no-80chars-check_ flag (1990 called, it wants its' 1024x768 resolution back).
2. Things you should know if you're maintaining modules
    * Use a module deployment manager.
        1. [r10k](https://github.com/adrienthebo/r10k). I now use this everywhere. You can easily set it up to deploy your modules from *master* to _/etc/puppet/environments/production_ and *dev* to _/etc/puppet/environments/development_ , which is so awesome. This makes staggering environments a snap. I plan on writing a quick guide to this soon.
        2. [Puppet libarian](https://github.com/rodjek/librarian-puppet) is another I've not yet used, but I've heard good and bad.
    * [Hiera](http://docs.puppetlabs.com/hiera/1/) is fantastic. Separate your data from your modules.
    * Use facts for advanced modules. Your modules should not try to determine how to run, but simply follow flow. For example, assume you have a module that should operate differently based on what's responding on an HTTP port. Your module should be as simple as possible, so it shouldn't try to determine this, but use it. You should write and package a *fact* that hits the server, and returns *my_http_server => 'nginx'*. Now your module can just do something as simple as a switch on this *$::my_http_server* variable. Done.
    * I love Craig Dunn's [roles and profiles](http://www.craigdunn.org/2012/05/239/) architecture. It's so simple and it works quite well. Having a singular *role* that maps to multiple *profiles* provides a flexible and easily maintained set of modules.
    * Puppet's [stdlib](https://github.com/puppetlabs/puppetlabs-stdlib) is great. I'm a huge fan of not reinventing the wheel, so having _str2bool()_ available makes me happy. Along with a ton of other well written functions you need.
3. Buzzwords (things I can't believe I've never heard of/used)
    * [Pulp](http://www.pulpproject.org/) is quite cool. For my modules, everything is RHEL/CentOS based, so publishing RPMs has become my 'thang'. Pulp *seems* to be an attempt to get spacewalk-like functionality into the hands of people who don't want the overhead of the database, GUI, and a bunch of the bulk. Me? I just use it as a Yum repository. My jenkins builds RPM's and publishes them to a Pulp server, so it becomes available to anything that uses it as a repo.
    * I've heard of and scoffed vim package managers. Until [neobundle](https://github.com/Shougo/neobundle.vim). I've been setting up so many servers lately, having a dotfiles repository with a .vimrc with all my Vim necessities has lowered my blood pressure.
    * [Packstack](https://wiki.openstack.org/wiki/Packstack). This could be a rant in and of itself. I really want to like this. I really do. What is it? It's a fully automated puppet setup of Openstack. What?! IaaS at your fingertips? Too good to be true you say? Yes. Great idea, but it needs a lot of work. I didn't sleep for 2 days trying to set this up. And gave up.
        * tl;dr *What is it*: Puppet based installer for a single instance version of Openstack. Openstack really requires 3 nodes (compute, data, network), but this gives you a 'trial version', meaning all 3 nodes on one instance with some limitations on Ram, storage, etc that you can't change (because why on earth would you actually want IaaS on a single instance).
        * tl;dr *My issues*: Failed on Fedora 19. And Fedora 20, although both are officially supported. Discovered bugs on each install such as the GUI sending only 500 ISE's, missing the x86_64 Qemu 1.4 emulator, and removing _/var/lock/subsys_ in an install.
    * [RDO](http://openstack.redhat.com/Main_Page) is *Red Hat*'s *D*eployment of *O*penstack - see what I did there? This is what I used to fail miserably at installing packstack.
    * [Foreman](http://theforeman.org) is a great and simple way to keep track of your hosts. Works flawlessly with your set up, whether you use an ENC, Hiera, both, neither. I use it and swear by it. A coworker loves the [puppet dashboard](http://puppetlabs.com/dashboard) although it seems to have the same functionality. I don't use either one for its provisioning, solely its puppet management.
    * [openshift](https://www.openshift.com/products/origin) seems to be red hats solution to heroku. PaaS still eludes me, but my understanding is it's an answer to developing without deploying. Push the code to their system with some metadata on how to execute and be done with it.

Random Things I Learned About Puppet Pre/Post Conference
========================================================

1. Puppet is the greatest thing I've found for linux system configuration and package maintenance....Windows? Not so much. Puppet is obviously intended for the linux community, seeing as the master HAS to be linux. This is fine if you're just wanting to set up and maintain a windows box. Where am I going with this? *Continuous deployment*. If you're looking to deploy code, continuously or not, on a Windows system, if it's not msi or exe, Puppet isn't your thing. I've found [octopus deploy](http://octopusdeploy.com/) and it works great from a CI/NuGet deployment standpoint.
2. [Puppet without root](http://lmgtfy.com/?q=puppet+without+root) is a pipe dream. It's a hot topic, but the biggest failure? Linux packages will most likely require root for a very long time. Oops.
3. Write facts, use hiera, and stop making complicated modules.
4. I love [vagrant](http://vagrantup.com/). So much.
