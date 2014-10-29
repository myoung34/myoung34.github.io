---
title: Yet another Puppet module - HBase
author: myoung
layout: post
comments: true
permalink: /post/hbase-puppet-module
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
  - hbase
  - nosql
  - git
---
In my current set-up I have now deployed a very ugly HBase RPM (0.96) with a puppet manifest to help other devs bring up hbase quickly as well as a more resilient version of the same setup to a coworkers hbase stargate project. I decided to sit down and make it "legit" by adding best-practices and real usage tests. <!-- more -->

I present yet another HBase module. Unlike the ones on the forge, however, this one works.

The RPMs are provided by a yum repo from [horton works data platform](http://hortonworks.com/hdp/) which are very well made, contain well-catered configuration files, and even uses zookeeper out of the box. If you have made it this far and wish to try it out, it is very simple and works exremely well. While I would NOT use it in production in favor of a clustered set up with Thrift, Region Servers, or even YARN, it will work for most dev set ups. If you use it in production, drop me a line on how well it performs under load.

If you are interested in something more suitable for production I recommand [Ambari](http://ambari.apache.org/). It's extremely awesome. Otherwise, just check out my current module at [my github](https://github.com/myoung34/puppet-hbase) or on the [puppet forge](https://forge.puppetlabs.com/myoung34/hbase).

Basic Usage:

```
class { 'hbase':
  port => '8080', #default is '8000'
}
```
