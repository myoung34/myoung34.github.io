---
title: Automatically deploy GIT code to Windows with Puppet
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/automatically-deploy-git-code-windows-puppet/
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
Recently I&#8217;ve been looking for ways of keeping a box up-to-date as far as git is concerned, and for windows the options are pretty limited. The vast majority of DevOps seem to like Puppet, so I decided to give it a try.<!--more--> The problem is windows isn&#8217;t very good without a few helper programs such as ssh, but we&#8217;ll get around that with Cygwin. The master box will be a Centos 6.4 x64 host.

## Puppet Master

The first thing we&#8217;ll need to do is install the EPEL and Puppet repositories with, then install the puppet software and start the master process: 

```
# rpm -Uvh https://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
# rpm -Uvh https://yum.puppetlabs.com/el/6/products/x86_64/puppetlabs-release-6-7.noarch.rpm
# sudo yum install -y puppet-server && chkconfig puppetmaster on && service puppetmaster start
```


Let&#8217;s go ahead and get some files prepared. Create a git repo in **/opt** called **test** with a file in it (**test.txt**), committed to master: 

```
$ cd /opt
$ sudo mkdir test && chmod 766 test
$ cd test
$ git init
$ touch test.txt
$ git add test.txt
$ git commit -a -m "Adding empty file"
```

 
One last thing we&#8217;ll need for puppet to work with git is VCSRepo, so let&#8217;s get that: 

```
# puppet module install puppetlabs/vcsrepo
```

 
Now that puppet is ready to serve it out, we have to create some files.

* **/etc/puppet/manifests/site.pp** 

```
node 'server-vm.blindrage.ad' {
  include git
}}
```


* **/etc/puppet/modules/git/manifests/init.pp** 

```
class git {
  class { git::clone: }
}
```


* **/etc/puppet/modules/git/manifests/clone.pp** 

```
class git::clone ($repo='dev', $username='marcus.young') {
  vcsrepo { "C:/${repo}":
    ensure   => latest,
    owner    => $owner,
    provider => git,
    source   => "git+ssh://${username}@foreman/~/repos/test",
    revision => $repo,
  }
}
```


    
## Puppet Client

The first two steps are going to be to install cygwin, making sure to add git and openssh, then the puppet client software (making sure to point to your linux host as the master). 

When this is done, stop the windows puppet service and make sure it is not set for startup. We&#8217;re going to control this process with a custom task schedule. 

Next, add the path to your cygwin bin folder to the PATH environment variable. If you don&#8217;t know how, it&#8217;s probably something similar to **C:\cygwin\bin**, and you&#8217;ll add it according to <a href="https://geekswithblogs.net/renso/archive/2009/10/21/how-to-set-the-windows-path-in-windows-7.aspx" target="_blank">this guide</a>. You&#8217;re close now! 

Using cygwin, generate a public/private key pair (if using git+ssh) **without a password** and give the public key to the master box. 

If you&#8217;ve done it right, you can clone the repo in cygwin without a password using this line: 

```
$ git clone git+ssh://{username}@{master}/opt/test
```

The last step is to generate a cert and sign it.  

From the client: 

```
puppet agent -t --waitforcert 60
```


Then on the master: 

```
# puppet cert sign --all
```

The client should now finish and download your git code! Congrats! 

However, this isn&#8217;t automagic yet. To do that, create a scheduled task that runs every 2 minutes (or whenever) that runs a daemon file.  
The easiest way is to create** C:\Program Files\Puppet Labs\Puppet\bin\puppet_daemon.bat** which contains: 

```
@echo off
echo Running Puppet agent on demand ...
cd "%~dp0"
call puppet.bat agent --test %*
```

Congrats. You&#8217;re done! To test, make a change to the git repo and wait two minutes!
