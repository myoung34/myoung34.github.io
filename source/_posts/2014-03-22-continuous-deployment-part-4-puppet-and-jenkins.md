---
title: Complete Continuous Deployment Using Puppet and Pulp - Part 4 - Puppet and Jenkins
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/continuous-deployment-using-puppet-and-pulp-pt-4-puppet-jenkins
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
So. We have an application that it's constantly pushed to Yum when it's modified. We have a puppet manifest (with tests) that can deploy this application and keep it up to date, and we even have r10k handling the deployment of those puppet modules so that we don't have to worry about dependencies, and we can even stage them! What we don't have is a way of getting changes to that puppet module to the puppet master without remoting in and running r10k. That's easy to fix with jenkins!<!--more-->

If you were to stop right now, you're pretty close. But...remoting into the server to redeploy isn't scalable. What if you forget? What if you have a new guy that made a super awesome bugfix to your puppet module and forgot how to get it up to the servers? Enter jenkins. Note: I have travis-ci set up for the one here, but we're going to duplicate that logic. Travis-CI is nice if your module is on github, but it may not be. Or you may need it to do something else, such as trigger an r10k deployment, so that's why we're still implementing Jenkins.

###The Jenkins job for puppet-exampleservice

  1. Create a free-style job and name it something like *puppet-exampleservice*
  1. Point it to your git repo for the *puppet-exampleservice* module and have it poll SCM every x minutes.
  1. Have it run in rvm 2.0.0
  1. Create an **execute shell** build step with:
```
bundle install
bundle exec rake
```
  1. From the *warnings* plugin, add a **post-build action** to *scan for compiler warnings*
    * set *scan console log* to **puppet-lint**

From here, you now have something similar to Travis-Ci. Hit Build, and it should go green. Congrats, you now have automated testing. What you need now is that automated deployment I've been hinting at. But that's easy!

###The Jenkins job to re-deploy puppet modules on the ENC

  1. Create a free-style job and name it something like *Redeploy Puppet Modules*
  1. Add a string parameter **git_branch**
  1. Add a build step **execute shell script on remote host using ssh**
    * Select your ENC (set one up in the **Manage jenkins** portion first and make sure your private/public keys work) with:
```
# Validate the build param. Should be 'dev' or 'master'
# This corresponds to the r10k setup.
environment=$(echo "$git_branch" | awk '{split($0,a,"/"); print a[2]}')
if [[ "$environment" == "master" ]]; then
  rm -rf /var/cache/r10k/* # Sometimes r10k won't deploy and acts like it's up to date.
  /usr/bin/r10k deploy environment master -v -p
  exit 0
elif [[ "$environment" == "dev" ]]; then
  rm -rf /var/cache/r10k/* # Sometimes r10k won't deploy and acts like it's up to date.
  /usr/bin/r10k deploy environment dev -v -p
  exit 0
else
  echo "Invalid environment parameter [$environment] passed to build."
  echo "  Should be one of 'dev' or 'master'."
  exit 1
fi
```

You're now ready for your previous module to notify this build.

  4. Go back to your puppet module job and:
    1. Create a post-build action **trigger parameterized build on other projects**
      * Set 'projects to build' to **Redeploy Puppet Modules**
      * Set 'Trigger when build is' to **Stable or unstable but not failed**
      * Add a parameter 'Predefined parameters' and give it: ``git_branch=$GIT_BRANCH``

### You're missing a Jenkins job

Technically you're close, but you might want to redeploy if your r10k changes. You can do that right now!

  1. Create a new free-style joba and call it *puppet-r10k*
  1. There are no tests, so just point it at your r10k git repo and watch *dev* and *master* branches
  1. Make it poll SCM every x minutes
  1. Create a post-build action **trigger parameterized build on other projects**
    * Set 'projects to build' to **Redeploy Puppet Modules**
    * Set 'Trigger when build is' to **Stable or unstable but not failed**
    * Add a parameter 'Predefined parameters' and give it:

###You're Done!!1one

Now build your puppet module. When it goes green, it will notify the **Redeploy Puppet Modules** job and give it the git branch (*dev* or *master*). That build will now parse the git branch, and on the Foreman ENC will run the r10k deployer. Don't you have that fuzzy feeling inside because you can now redeploy your modules to the server by just making a commit? I do and I don't even know you!
