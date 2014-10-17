---
title: '(FIX) MediaTomb/uShare Won&#8217;t Broadcast SSDP'
author: myoung
excerpt: "This is a quick guide to try and list common fixes for devices like the ps3 and xbox 360 that can't see linux dlna/upnp servers like MediaTomb and uShare. "
layout: post
comments: true
permalink: /post/fix-mediatombushare-broadcast-ssdp/
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - Fix
  - Linux
  - Media
tags:
  - 360
  - dlna
  - igmp
  - iptables
  - ports
  - ps3
  - selinux
  - ssdp
  - upnp
  - xbox
---
If you&#8217;re running Mediatomb or uShare and your xbox, ps3, tv, or whatever other UPnP/DNLA device doesn&#8217;t see the server, here&#8217;s a few fixes I&#8217;ve discovered.<!--more-->

* If you&#8217;re running SELinux, disable it temporarily (it doesn&#8217;t play well sometimes)
* Both of these use the IGMP protocol to display their SSDP. Try allowing it through IPTables: {% codeblock bash %}# iptables -A INPUT -i <em1|eth0|device name> -s 0.0.0.0/32 -d 224.0.0.1/32 -p igmp -j ACCEPT{% endcodeblock %}
* Make sure your router allows UPnP
* Ensure the right ports are allowed through the firewall 
  * ushare
    - UDP 1900
    - TCP 2869
    - TCP 5000
  * mediatomb
    - The port you have set in the /etc/mediatomb.conf as MT_PORT

I spent quite a while trying to figure this out on my set-up, and it turned out the igmp fix was all I needed. 
Days of work for 1 line in my /etc/sysconfig/iptables file.
