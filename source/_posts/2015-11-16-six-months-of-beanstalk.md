---
title: My Thoughts after six months of Elastic Beanstalk.
author: myoung
layout: post
comments: true
permalink: /post/six-months-of-beanstalk
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - aws
  - beanstalk
  - ruby
tags:
  - aws
  - beanstalk
  - ruby
---

We've been using pure beanstalk at my current gig for six months and here are my thoughts. <!-- more -->

I'll start off with the tl;dr: I like beanstalk a lot. When it fits.

I've written enought deployment mechanisms to go a little crazy in my time. From tagging code in SVN and pushing it to a central store, to purely puppet provisioned instances, to microservices with ad-hoc AMIs to beanstalk. And beanstalk works. If you have a *traditional* application (by that I mean some consumers, some APIs, and a database of some sort), beanstalk is the (insert here). 

Our current stack includes:

 * A backend rails API.
 * A front-end javascript CMS.
 * An ad-hoc CRUD API written in rails.
 * A javascript web-embed written in node.


We use jenkins to deploy code changes once tests pass.

And I've got to say: I'm not unhappy. There are very very few tweaks to make any more. 

 * All of these have multiple environments (dev/uat/prod), of which dev and uat shut down at night and on weekends.
 * Deployment includes rollbacks and notifications on failure.
 * Things auto-scale as expected.
 * You get all the metrics you really care about out of the box.

That said: if it doesn't fit your stack, don't hamstring it. There are many good deployment mechanisms out there, and if your application does not easily fit into the peg that is beanstalk: find something else. We attempted to use it as a deployment mechanism for our cache layer and it failed horribly. Was it because there is no built in support in beanstalk? Yes. But It would be really nice to see beanstalk provide a boiler plate type set up like a template. Basically you just use their default AWS Linux image with no framework on it, provide the deployment scripts, and be able to manage it via beanstalk. That would be fantastic. 

The only downside that I have found is that I would really like to re-do our base architecture at AWS to be code-based.
I'm pretty much sold on Terraform, but that is my personal choice, and I would really love to see first-hand beanstalk support there. While not a criticism of beanstalk, I'll just be happy and watch [this issue on terraform about the topic](https://github.com/hashicorp/terraform/issues/2799).
