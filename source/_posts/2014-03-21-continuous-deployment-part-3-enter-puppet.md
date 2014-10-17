---
title: Complete Continuous Deployment Using Puppet and Pulp - Part 3 - Enter Puppet
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/continuous-deployment-using-puppet-and-pulp-pt-3-enter-puppet
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
Now that we have a package, we need a way of installing it amirite? We're going to write a puppet module for this package complete with rspec tests<!--more-->

# The Module

The module for this is going to be simple. The application as it is right now is an HTTP server in python. So our module needs to do these things:

  1. Manage the package and make sure it's always the latest
    * This ensures that new builds will be reinstalled
  1. Open the firewall for TCP port 8000
  1. Make sure dependencies are installed
    * The RPM has a dependency set for *python*, so yum would take care of this, but I like to be explicit and not assume that the RPM knows edge cases.

## puppet-exampleservice breakdown

### Files

  * *manifests/init.pp*
    * This just kickstarts the puppet catalog for this module. It will call our *exampleservice.pp* class
  * *manifests/exampleservice.pp*
    * This is the hard worker. It will make sure the service is running, the package is the latest, and that the firewall is open. This implies a dependency on the puppetlabs-firewall module. 
  * *Puppetfile*
    * This will let you know dependencies, such as the *puppetlabs-firewall* module
  * *.travis.yml* and *.fixtures.yml*
    * These are specific to [travis-ci](http://travis-ci.org). If your module will be internal, and not at github, you won't need this. I did so because it's at github. I'll be duplicating this CI at jenkins, and you'll see why later.
  * *Rakefile*, *Gemfile*, and *spec/*
    * These are all part of the testing framework. The tests included for this module are **Puppetlint** (with some disabled checks) and **Rspec-Puppet**
 
### Tests

  1. Puppetlint
    * Checks syntax and best practices for your modules. Highly recommended. To make use of this in jenkins, you'll want to take a look at [this guide](http://hackers.lookout.com/2012/07/puppet-lint-with-jenkins/)
  1. RSpec-Puppet
    * Let's you write assertions for the catalog. Examples would include testing manifest logic, like not doing something if a flag was true or false, etc. See [the rspec-puppet pages](rspec-puppet.com/tutorial/)

# Making your module work via r10k

You have a module. How do you get it on foreman, or any Puppet master to actually work? Are you going to remote into it, check it out to some repository folder, symlink it to **/etc/puppet/modules** and call it good? That'll work, but how do you manage dependencies? Remember, you need the *puppetlabs-firewall* module there too for this to work. You'll probably need others. What happens when you have to fix this module? Fix it, see the tests are green (or dear god fix it on the ENC), and call that your process? The point of continous deployment, devops, etc etc is to have **consistency**. Having a one-off process for deploying your configuration management violates what you're trying to solve! Enter **r10k**. Essentially all you need is a new git repository (call this one [blog-r10k](https://github.com/myoung34/blog-r10k) that holds information about what modules you need, where they came from, and what versions you want of them and where. This also adds the ability to **stage** modules. You can have a *dev* branch for your module, deploy it to *development* nodes, and vice versa. This means you can stop hoping your fix works and prove it before it hits production boxes.

In the [Part 1](/post/continuous-deployment-using-puppet-and-pulp-pt-1-the-machines/) I hinted at modifying the *[development]* and *[production]* labels in the puppet configuration file. Now is when you find out why.

### Set up r10k

On your ENC, create a file **/etc/r10k.yaml** with this content:

```
:cachedir: '/var/cache/r10k'
:sources:
  :puppet:
    remote: "https://github.com/myoung34/blog-r10k.git"
    basedir: '/etc/puppet/environments'
:purgedirs:
  - /etc/puppet/environments
```

### Deploy your modules the 'hot' way

How? On the ENC type: ``# r10k deploy environment -v -p``

###What wizardry is this?!

That r10k file tells you to purge */etc/puppet/environments* and deploy the *Puppetfile* from the [r10k repo](https://github.com/myoung34/blog-r10k/blob/master/Puppetfile) in a similar fashion to Gem, NuGet, etc. You told it to deploy all environments it knew about to that directory. The output showed a **dev** environment and a **master** environment. These are git branches! Each branch in the [r10k repo](https://github.com/myoung34/blog-r10k/blob/master/Puppetfile) has a Puppetfile. You'll notice that in the master branch, it said to deploy *puppetlabs-firewall* and *puppet-exampleservice*. Look in the dev branch of the [r10k repo](https://github.com/myoung34/blog-r10k/blob/master/Puppetfile). The Puppetfile says to use ``:ref => 'dev'`` for your *puppet-exampleservice*. You just staged your code. Your **dev** code now deploys to **/etc/puppet/environments/dev/modules** and your **master** code now deploys to **/etc/puppet/environments/master/modules** ! Remember the change you made to your puppet.conf? Any node that has ``environment = development`` will generate its catalog using **/etc/puppet/environment/dev/modules** and ``environment = production`` will generate its catalog using **/etc/puppet/environment/master/modules**.

###Why you care even if you think you don't

That passive aggressive statement I had criticizing your workflow above is now null. If you make a change to your module in **dev**, all you have to do is remote into your puppet master where *r10k* is, and type ``# r10k deploy environment dev -v -p``. No more missing dependencies, it's hands off!

### What's missing

  1. Acceptance tests
    * Before this would have been **rspec-system-puppet** but it's deprecated for [Beaker](https://github.com/puppetlabs/beaker). Note that both of these use Vagrant to spin up a siloed Virtual Machine to test the module in isolation. If you're wanting to do this, make sure your jenkins can spin up a virtual machine. If your jenkins is at AWS for example, you'd probably want to look at having it spin up an AMI to run this. My current setup does not allow the creation of VMs from a VM (My jenkins is a VM).

##What you have at this point in this blog post

Right now, you have a Jenkins box that generates new RPMs of your application whenever you make changes to it. You also have a manifest that can manage it. However, this is a one-off. If you have to modify your manifest, you have to redeploy it via r10k to the ENC (foreman). It would be nice if you could utilize Jenkins for this. That's in the next post.
