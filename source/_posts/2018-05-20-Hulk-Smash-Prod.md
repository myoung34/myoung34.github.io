---
title: Hulk-Smash Production
author: myoung
layout: post
comments: true
permalink: /post/hulk-smash
image:
  -
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - production
  - hardware
  - python
  - mechanical keyboard
tags:
  - production
  - hardware
  - python
  - mechanical keyboard
---

I one-upped my own bad idea with more badness. Don't do this.  <!-- more -->

I, uh. Yeah.
 
I assume you [read my previous post about my physical deploy button](/post/deploy-production-button). Well: I one-upped it.

Let me explain. Nah let's cut to it. I wanted to play with [AWS IoT](https://aws.amazon.com/iot-core/) and I did so shamelessly with much architecture.

### The architecture

I had a spare Raspberry Pi Zero W with a camera. So I did what any reasonable person would do. Used it with no end-goal.

I hooked it up to AWS IoT and made it [listen to the shadow delta MQTT queue](https://docs.aws.amazon.com/iot/latest/developerguide/using-device-shadows.html)

Basically whenever it saw a change to the shadow it would:

 1. Spin a [stepper motor](https://www.amazon.com/Stepper-Bipolar-4-lead-Connector-Printer/dp/B00PNEQKC0) that landed a [hulk hand](https://www.amazon.com/Marvel-Avengers-Gamma-Grip-Fists/dp/B072QMZTZ4) on my deploy production button.
 2. Thanks to py3 [threads](https://docs.python.org/3/library/threading.html) it would also start the rasp pi camera to watch the hulk smash and send it to [giphy](https://www.giphy.com).
 3. After the gif is done uploading it would send a gif of the hulk smash back to the user in slack.

### The finale

Without further ado I give you: Hulk Smash Production.

{% img /images/hulksmash.png hulksmash %}

The [initial gif](https://giphy.com/embed/xuW89v9kQXMeQ)

And the final gif (yes the hulk hand is duct taped to a bamboo skewer to a stepper motor):

{% img /images/hulksmash.gif hulksmash %}
