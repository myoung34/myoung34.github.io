---
title: A Physical Deploy-Production Button
author: myoung
layout: post
comments: true
permalink: /post/deploy-production-button
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

I made a physical button to deploy production and regret nothing.  <!-- more -->

Over the last few weeks I've been looking for a project.

I currently work for a python shop and have really taken a liking to it. Simultaneously, I keep seeing new hardware [supporting micropython](https://micropython.org). I'm not against the C'ish language that the arduino type stuff pushes for, but it's nice to get the best of both worlds: an abstracted language with nice syntax and the ability to do dumb stuff with voltage. So: I did just that.

### The Controller

I've really wanted to play with the ESP8266 after seeing some cool projects cross over my feeds. I decided also to kill two birds with one stone and try the Huzzah board [from adafruit](https://www.adafruit.com/product/2821) that includes onboard wifi because I hate money. The board itself was fantastic out of the box. 

### The Learning Curve

Without doing any whatsoever much research my brain really thought that micropython was going to be a "drop in replacement" for the UX of programming a board in the arduino IDE. It's not. This is actually my biggest complaint about micropython. The code itself was very very very straightforward. Write a `main.py` that does your stuff, and profit. You can load libraries that support micropython, etc. 

Getting there? Meh.

First: you have to flash your esp with [the micropython firmware](http://micropython.org/download). 

Next: you have to get your code onto the board. The feedback loop for this is the annoying part. The first time you flash the firmware you have to [enable WebREPL](https://learn.adafruit.com/micropython-basics-esp8266-webrepl/access-webrepl). It's a one-time cost but it's smelly. This enables a wifi broadcast from the board that you can then access from webrepl with a password. 

If you make it past the previous step you can make `main.py` join your wireless network (note: 5ghz is not supported) with something like:

```
import network

sta_if = network.WLAN(network.STA_IF)

def do_connect():
    if not sta_if.isconnected():
        print('connecting to network...')
        sta_if.active(True)
        sta_if.connect('The LAN Before Time', 'hunter2')
        while not sta_if.isconnected():
            print('waiting to connect...')
            sleep(5)
            pass

print('connecting...')
do_connect()
print('network config:', sta_if.ifconfig())
```

Now that it's on the network you can use webrepl to the local address of the huzzah. But it still sucks compared to the typical UX of: select board, click compile.

### What You Came To See

If you're still reading I'm sure you want to see what I did. Basically I took 4 mechanical keyboard switches (I have a lot OK?!), wired them to the huzzah, and made it send a slack command to our internal slack bot to *force* deploy production (ignore all locks, send a message to yours truly).

```
while True:
    buttons = [
        machine.Pin(4,  machine.Pin.IN, machine.Pin.PULL_UP),
        machine.Pin(5,  machine.Pin.IN, machine.Pin.PULL_UP),
        machine.Pin(2,  machine.Pin.IN, machine.Pin.PULL_UP),
        machine.Pin(15, machine.Pin.IN, machine.Pin.PULL_UP),
    ]

    _sum = 0
    for button in buttons:
        _sum += button.value()
    if (_sum == 0):
        print('generating request')
        print(urequests.post(webhook_url, json={'text': ":alert: DEPLOY BUTTON PRESSED. KLAXON: ACTIVATED :alert:"}, headers={'Content-Type': 'application/json'}).status_code)
        print(urequests.post(webhook_url, json={'text': "!deploy prod master -f"}, headers={'Content-Type': 'application/json'}).status_code)
```


{% img /images/huzzah1.jpg huzzah1 %}
{% img /images/huzzah2.png huzzah2 %}

It works and it's pretty empowering to smash. We deployed to production a record number of times that day.
