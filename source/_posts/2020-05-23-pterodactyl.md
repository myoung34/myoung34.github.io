---
title: Pterodactyl - The bluetooth dactyl fork
author: myoung
layout: post
comments: true
permalink: /post/pterodactyl
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - keyboard
  - ergo
  - keyboard
tags:
  - keyboard
  - ergo
  - dactyl
---

I spent far too much time making a bluetooth dactyl keyboard with built in wrist rests <!-- more -->

This took about a year, off and on. With covid I finally found some time to finish it.

### Build Hardware ###

* Base was customized heavily by me from [this one](https://www.thingiverse.com/thing:2436848). [Mine here](https://www.thingiverse.com/thing:4186032) supports less ugly wrist pads IMO, printed in Wood PLA from hatchbox on a taz6 at 0.1micron height
* [Feather 32u4](https://www.adafruit.com/product/2771)
* TRRS cable
* [MCP 23018 io expander](https://www.mouser.com/ProductDetail/Microchip-Technology/MCP23018-E-SP)
* [Wrist rests](https://www.amazon.com/gp/product/B07DF83HK7)
* [TMK Oblotzky Oblivion V2 caps](https://drop.com/buy/drop-oblotzky-gmk-oblivion-v2-custom-keycap-set)

### Notes ###
I ran into a lot of issues.

1. With the feather I had to swap `GPIOA` and `GPIOB` for `EXPANDER_COL_REGISTER` and `EXPANDER_ROW_REGISTER`
2. It took *months* for me to realize that the Serial stuff with `SPLIT_KEYBOARD` [here](https://beta.docs.qmk.fm/using-qmk/hardware-features/feature_split_keyboard) does *not* work with an IO expander like the MCP23018. This is poorly documented IMO. You have to use two of the same chipsets to use that feature and the feather 32u4 aint cheap
3. [this issue](https://github.com/adereth/dactyl-keyboard/issues/57) would have been massively helpful originally to know I was wiring it incorrectly.
4. Swapping the board from the right side to the left side turned out massively tedious and [my PR](https://github.com/qmk/qmk_firmware/pull/9181/files#diff-6b1633df300bad911cd57febbbbdb2aaR63) suffers for it.
5. Im dumb and bought a mislabeled TRS cable as a TRRS cable and it obviously didn't work combined with my inability to count the rings on it.

### Pics or GTFO ###

{% img left /images/ptero1.jpg %}
{% img left /images/ptero2.jpg %}
{% img left /images/ptero3.jpg %}
{% img left /images/ptero4.jpg %}
{% img left /images/ptero5.jpg %}
{% img left /images/ptero6.jpg %}
{% img left /images/ptero7.jpg %}
{% img left /images/ptero8.jpg %}
