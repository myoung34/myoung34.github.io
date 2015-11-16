---
title: Fighting a nasty AWS bug for Beanstalk Leader Election
author: myoung
layout: post
comments: true
permalink: /post/beanstalk-leader
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

In the last few months I've been working to move my latest employer to AWS from Heroku and discovered a very nasty bug. <!-- more -->
The move was due to a lot of issues, the least of which being the cost of heroku compared to using what they implement.

None-the-less, everything went smoothly and the API went from roughly 80% uptime to 99.9%. 
The cool thing: we used timed auto-scaling to shut down our dev/test instances at night time and on weekends. This saved *alot* of money. (remember this part).

After about two months, we discovered migrations in our rails application stopped. The code would deploy fine, but migrations just didn't. After much trial and error
it turned out to only happen on dev and test (not production). Parsing the web of beanstalks deployment framework, it turns out they have the concept of a leader. This is good. With the concept of a leader, you ensure that you can deploy to fleets in bulk and things that are stateful will only be run once (on the leader). One of these things is rails migrations, which makes perfect sense.

Somehow, the idea of our leader was lost. Out of all of our instances (2 in dev), neither one was considered a leader. Dropping that to a single instance, you would think: there's your leader right there. Nope. 

Through much debugging with AWS support, we discovered that the implementation is based somewhat loosely [on this](https://github.com/blake-education/aws-cfn-bootstrap/blob/master/cfnbootstrap/cfn_client.py).

Turns out that a leader is elected once in the lifecycle of a beanstalk *application* and is passed around as the instances scale. If all instances are lost simaeltaneously, that leader is gone, and that's the bug. This could be problematic if you're not careful, but beanstalk makes it stupid simple to make sure instances are almost always available. It would take something crazy to lose all at once. 

Like shutting down all instances at night.

As of now, the workaround has been to set an environment variable `EB_IS_COMMAND_LEADER` to `true`, and set our deployment to 1 at a time. While it works, it ties us back to beanstalks' internalness and makes our deployment much slower. Not only because it deploys to one at a time, but because each instance is going to run migrations even when not necessary.

Update: they've fixed it with their latest update as of 11/15/2015. I removed the environment variable and deployed to a fleet, and it behaved as expected
