---
title: Making my own plaato keg firmware
author: myoung
layout: post
comments: true
permalink: /post/plaato_keg_esphome
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - homebrew
  - hacking
  - esphome
tags:
  - homebrew
  - hacking
  - esphome
---

After two years of failing to deliver a promised feature I decided to reverse engineer and make my own plaato keg firmware <!-- more -->

<br/>
<br/>
I backed this project because plaato makes really good quality stuff. The idea of having a weight based keg estimator for my barcaderator was key.
They promised multiple times they'd allow people to capture the data themselves. They have yet to deliver.
<br/>
<br/>

{% img /images/keg/keg6.png %}
<br/>
```
Andreas (PLAATO)

Aug 12, 2020, 3:14 PM GMT+2

Hi! We've unfortunately had to put the webhook for PLAATO Keg
on the back burner as we are working on other projects, 
particularly with our backend, that take priority at the moment. 
While we would very much like to implement the webhook we 
unfortunately don't have any ETA on this as of now.
```
<br/>
```
Abi from Plaato

Feb 11, 2022 2:21 AM GMT+2

Hey Mark,

Thank you for sharing. What I will do to make sure your voice is 
heard is to share your message as a Feature Request to the development
team.

I hope this helps, please let me know if you have any additional 
questions.
```

So I decided: why not do it myself?

In january 2020 I did a <a href="/post/homebrew-hacking/">PyTN talk about how to sniff the data.</a> While cool, it didn't really bear much fruit. The amount of effort to make this be my end game was a bit much and I'd be reliant on always intercepting it. I did however know that its got an ESP32-D0W0 (wroom) as the brains and it didnt look that complicated.

So I took it down again and flashed a basic esphome build to it (the code is <a href="https://github.com/myoung34/plaato-keg-esphome">here.</a>)

<br/>
{% img /images/keg/keg.jpg %}
<br/>

This was easier than it looks, you just need a steady hand. Once you flash it the first time you can do pure OTA updates.

I wont go into absolute detail but you can just look at the pinout for the ESP32 and solder leads to the pins directly. RX, TX, Gnd, Vcc, and GPIO (grounded) directly to an FTDI programmer. 

First I made a backup of the original firmware:

```
esptool.py --chip esp32 --port /dev/tty.usbserial-00000000 \
  -b 115200 read_flash 0 0x400000 plaato_backup.bin
```

Then I flashed the new one built from ESPhome:

```
esptool.py --port /dev/tty.usbserial-00000000 write_flash \
  --flash_mode dio --flash_freq 40m --flash_size detect 0 \
  plaato.bin
```

Now it shows up in ESPHome as expected!

I then used a (very old) osciloscope to try to map out the load cells (load cell combinator is pretty boring (yayyy) and just drags the voltage, ground, data, etc over a ribbon cable to the main PCB. This main PCB uses a `P##` notation which are not 1:1 with the ESP32 (booooo)


<br/>
{% img /images/keg/keg1.jpg %}
<br/>

I was able to discover that the clock for this is P14. I then used a multimeter to correlate P14 to GPIO16 (by dragging the other load across all the ESP32 pins until a beep gave it away). I did this to discover the data pin from the load cell combinator is P28. My handy dandy Multimeter let me correlate P28 to GPIO 17 on the ESP32.

<br/>
{% img /images/keg/keg2.jpg %}
<br/>
<br/>
{% img /images/keg/keg3.jpg %}
<br/>

I was able to figure out the LEDs without an oscilloscope by just using a multimeter. The main PCB just uses GPIO to turn those pins high or low to run the 3 LEDs, which is boring (yay).

The rest was trial and error.

Next I used [zapier webhooks](https://zapier.com/page/webhooks/) to receive the [HTTP Post](https://esphome.io/components/http_request.html) from esphome to send it to Zapier, which then sends it to datadog.
<br/>
{% img /images/keg/keg8.png %}
<br/>
<br/>
{% img /images/keg/keg7.png %}
<br/>

Check out the <a href="https://github.com/myoung34/plaato-keg-esphome">repo for my up to date esphome yaml</a>
