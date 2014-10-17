---
title: Complete Continuous Deployment Using Puppet and Pulp - Part 2 - The Application
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/continuous-deployment-using-puppet-and-pulp-pt-2-the-app
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
Our application will be simple, as its goal is only to demonstrate a daemon that needs to be managed via init.d and packaged as an RPM.<!--more-->

#The Application

Like I said, it's simple. It's going to be a shell script that runs the *python SimpleHTTPServer* on port 8000, but it brings up some real life issues:

  * How can we package it
  * How can we ensure the port is open
  * How can we test it

The code is on [this github page](https://github.com/myoung34/blog-exampleservice).

You'll notice the layout mimics its place on the filesystem. This may not be a realistic case, but it sure make sthe POC easy. You'll notice a **Rakefile** in the repository. This is the heart of our packaging for this. If you look into it, you'll see it's essentially a Makefile (see what I did there???). There are many other ways to approach this, but it's the simplest for now. Basically the rake file will parse the *version* file (which is [semantically versioned](http://semver.org/)), and tries to build an RPM via FPM based on it.

  1. The **unstable** build will happen as ``bundle exec rake unstable``
    * This will create an rpm such as: **myapp-0.0.1-beta.1.{timestamp}.x86_64.rpm**
    * This is timestamped because many merges to dev can happen before a version bump.
  1. The **stable** build will happen as ``bundle exec rake stable``
    * This will create an rpm such as **myapp-0.0.1-1.x86_64.rpm**
    * There should be no timestamping here, master merges should always be tagged and versioned appropriately.
  1. Both of the previous builds will attempt to upload to the pulp repository we set up.

#The Jenkins set up
### There is a pre-requisite.

If you look at the Rakefile closely, you'll see (or have wondered already) that pulp-admin requires a username and password. You wouldn't want to put this in the git repository, so the Rakefile has an implicit requirement for a file **.pulp.yml** in the home directory (for Jenkins this should be */usr/lib/jenkins*.

### The Jobs
  1. Unstable Job
    * Create a job that watches that git repository's **dev** branch
    * The git repository is https://github.com/myoung34/blog-exampleservice.git
    * Poll SCM every 3 minutes ``H/3 * * * *``
    * Run under rvm. I chose **2.0.0**
    * Run a build script:
```
bundle install
bundle exec rake
```
  1. Stable Job
    * Create a job that watches that git repository's **master** branch
    * The git repository is https://github.com/myoung34/blog-exampleservice.git
    * Poll SCM every 3 minutes ``H/3 * * * *``
    * Run under rvm. I chose **2.0.0**
    * Run a build script:
```
bundle install
bundle exec rake stable
```

# Profit

You'll notice that each time you make a change to the git repository, it will build an RPM for it and upload it to the Pulp server. That means that if you're subscribed to that Yum repository, you could do a ``yum update myapp`` and you'd get a new version. We're getting very close to done =)

{% img left /images/jenkinsbuild.png jenkins %}
