---
title: New Puppet Module - MirthConnect
author: myoung
excerpt: "test"
layout: post
comments: true
permalink: /post/mirth-puppet-module
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
  - hl7
  - mirth
  - git
---
I released a new Puppet module to the forge out of another work necessity. Mirth is used pretty heavily for HL7 ETL, and has a ton of other uses. <!-- more -->

I started out at my current gig doing this, so it hurt a little to have to automate it, mostly out of fear of getting back into that engine =).

It can be found [on my github](https://github.com/myoung34/puppet-mirthconnect) or at the [Puppet forge](https://forge.puppetlabs.com/myoung34/mirthconnect).

Basic Usage:

```
yumrepo { "my mirth repo":
  baseurl  => "https://server/pulp/repos/mirthconnect/",
  descr    => "My Mirth Connect Repository",
  enabled  => 1,
  gpgcheck => 0,
}

class { 'mirthconnect':
  provider => 'yum', #default is 'rpm'
}
```
