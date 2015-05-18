---
title: Azure, Github, and a Corporate Proxy
author: myoung
layout: post
comments: true
permalink: /post/azure-github-proxy
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - cloud
  - azure
  - .net
tags:
  - cloud
  - azure
  - github
  - asp.net
  - .net
---

Recently I started playing around with Azure and deploying an ASP.NET MVC site via its Github linkage. I had some strange issues.<!-- more -->

Problem 1: I am a masochist
===========================

Under the strange suspicion that I had no idea how to actually build NuGet packages and deploy an ASP.NET site via CI/CD I decided to take on the masochistic task upon myself.
If you [take a look at my very basic MVC site](https://github.com/myoung34/HelloWorldASP), you won't see much of interest. None actually, I'm not sure why you possibly clicked that link.

It's a basic MVC app using a terrible NuGet package I've been deploying that adds some string extension methods. If you run that in Visual Studio, all is well.

Problem 2: The work proxy
=========================

I tried to publish it to Azure using [the publish feature from VS2013 into Azure](http://blogs.technet.com/b/cbernier/archive/2013/09/24/deploy-your-web-application-to-windows-azure-from-with-visual-studio.aspx) but immediately ran into a 'me' problem. 

You see, I was deploying from my work PC, which is absolutely blocked from doing anything. All of the traffic from my work PC **must** be published through a web proxy on-site. 

This is fine for things like Chrome, which pick those settings up automatically. For cygwin (see problem 1) there are a lot of steps to take.

* wget/Vagrant: I had to export ``http_proxy``, ``https_proxy``, and ``ftp_proxy`` (after pulling down the proxy PAC and finding which proxy address to use).
* cURL: I had to write a wrapper ``/usr/local/bin/curl``(higher in the PATH) that basically said ``/usr/bin/curl --proxy 'http://proxyaddress:80' "$@"``.
* subversion: I had to add ``http-proxy-host = proxyaddress`` into ``~/.subversion/servers`` in the ``[global]`` section.
* git: OMG GIT. So here's the thing. git behind a corporate firewall is hard. It can't resolve through DNS and it can't do anything it needs to do. 
  * First I installed [ntmlaps](http://ntlmaps.sourceforge.net/) into ``~/software/ntlmaps``
  * Next I set my ``~.bash_profile`` to start up a command in screen if it hasn't already to start ntlmaps.
    * ``[[ -n $(screen -ls | grep -E 'git-proxy.\*\((A|De)tached\)$') ]] || screen -d -S git-proxy -m python ~/software/ntlmaps/main.py``
  * Next I set git to use this proxy: ``for i in http https; do git config --global $i.proxy http://localhost:5865``

Problem 3: Visual Studio and Azure Deploy
=========================================

[Visual studio cannot deploy to Azure behind a PAC proxy](http://blog.kloud.com.au/2014/11/13/publish-to-a-new-azure-website-from-behind-a-proxy/). 

Sites like this one claim you can make changes to the VS2013 executable config but I had zero luck. 

So little luck in fact that it just plain failed with an unconnectable error.

From then I said nevermind and decided to just link my Azure app to my github repo using the web UI. All went great and it immediately deployed however when I browsed to my website I was [greeted with this error.](http://www.dotnetthoughts.net/azure-system-methodaccessexception-attempt-by-security-transparent-method/).

All of the blog posts and stack overflow questions say to install the WebHelpers NuGet package via ``Install-Package -Id  Microsoft.AspNet.WebHelpers``, but that didn't solve it. It redeployed and had the same error.

The posts also say to just select the ``Remove additional files at destination`` checkbox on Visual Studio deploy. Well that's useless because I cannot deploy via visual studio. After fighting long and hard and redeploying via Github repeatedly, I found the answer.

Answer: Select the ``Remove additional files at destination`` no matter what.
=============================================================================

Yup. I changed to the guest network, selected the checkbox, published successfully and all went well. My app was now live.
I then pushed multiple changes/rebases into the github repo, and it's been working fine since.
