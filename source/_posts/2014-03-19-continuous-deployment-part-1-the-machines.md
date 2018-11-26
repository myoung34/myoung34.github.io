---
title: Complete Continuous Deployment Using Puppet and Pulp - Part 1 - The Machines
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/continuous-deployment-using-puppet-and-pulp-pt-1-the-machines
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
You're going to need a minimum of 4 servers. Before you freak out, that's to keep things simple. Again, calm down. Also, you'll see I like CentOS. All the servers I built were CentOS 6.5 x86_64<!--more-->

#Servers
## Git server
For this you don't really need to set this one up, but most likely you'll have an internal git server. I'll be using Github for this demo. Use whatever you like.

## Pulp
Set up the server and client via [this guide](https://pulp-user-guide.readthedocs.org/en/pulp-2.3/installation.html).

When done, go ahead and create an *unstable* feed via:

```
pulp-admin login -uadmin -padmin
pulp-admin rpm repo create --repo-id=unstable \
  --relative-url=unstable --serve-http=true \
  --display-name='Unstable Packages'
pulp-admin logout
```

## Jenkins
Follow [this guide](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+on+RedHat+distributions) to set up jenkins.

Install the jenkins plugins:

  * git scm
  * ssh plugin
  * rvm
  * warnings

Install these extra Yum packages:

`` yum install -y rpm-build rpm ruby ruby-devel rubygems git automake autoconf gcc gcc-c++``

[Set up RVM](https://rvm.io)

Install [FPM](https://github.com/jordansissel/fpm) via gem: ``gem install fpm``

Install the pulp-admin tools the same way you did on the pulp server **Not the server, just the admin tools!**

Go ahead an ensure all these bits and pieces work, such as connecting to pulp via the login command, using rvm, etc. Jenkins is the heavy lifter in this stack.

## Foreman
[The guide for 1.4.1 is here](https://theforeman.org/manuals/1.4/quickstart_guide.html#Installation)

Install **r10k** via gem: ``gem install r10k``

Also, install the Puppetlabs yum repository and update puppet to 3.4.2 or higher.

Updating Puppet from 2 to 3 on Foreman requires some configuration changes. To get these, just re-run the ``# foreman-installer`` command.

Make these changes to **/etc/puppet/puppet.conf** for now:

  * Under the **[master]** section add: ``hiera_config   = $confdir/hiera/master/hiera.yaml``
  * Replace the **[development]** and **[production]** elements at the bottom with:
```
[development]
    modulepath     = /etc/puppet/environments/dev/modules:/etc/puppet/modules
    config_version =
[production]
    modulepath     = /etc/puppet/environments/master/modules:/etc/puppet/modules
    config_version =
```

## A puppet node
Because why not. Install the puppet yum repository and install Puppet 3.4.2 or higher.

Make sure you have ``pluginsync = true`` in the **/etc/puppet/puppet.conf** file under **[main]**

My assumption before you move on, if you're following along, is that this node can connect to the puppet master (foreman) and generates all green when you run ``# puppet agent -t``

Lastly, make sure you're subscribed to the Yum repository for the **myapp** package we've been building so that puppet can install it when we get to that point. To do that, create the file **/etc/yum.repos.d/mypulp.repo** with the contents:

```
[mypulp]
name=My Pulp Provider - Unstable
baseurl=https://pulp.blindrage.local/pulp/repos/unstable/
enabled=1
gpgcheck=0
metadata_expire=10
```
