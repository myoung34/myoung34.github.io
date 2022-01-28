---
title: My Secure Smart Home
author: myoung
layout: post
comments: true
permalink: /post/smart-home
image:
  -
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - python
  - pytn 
  - conference
  - hacking
tags:
  - python
  - pytn 
  - conference
  - hacking
---

A quick write up of how I have IoT devices that I consider secure (at the cost of my sanity).  <!-- more -->

I've spent the last year figuring out what I want to control "things" in my house. And after wasting a bunch of time and money I finally have a set up I like.


## The Network ##

~~I don't have any crazy networking gear, just a [Netgear r7000](https://www.amazon.com/R7000-100PAS-Nighthawk-Parental-Controls-Compatible/dp/B00F0DD0I6) running [Kong](https://www.myopenrouter.com/kong-downloads)~~

I now have a very different set up.
{% img /images/router.jpg %}

Configuring this for my setup is painful, but:

* Main Network (wired + wireless N+AG): I whitelist all my devices by Mac Address and assign static IP's to things.
* Guest Network: no whitelisting, can only reach the internet
* Black Hole Sun: My black hole wifi for things I don't trust like my TV. I know for a fact that the TV tries to send data over open wifi's so my hope here is to prevent that by giving it valid networking that just cant talk to anything at all.
* Server network: My main server stuff is here. A nomad cluster across an i7 server and a bunch of pi's, also a [DS918+](https://www.synology.com/en-us/products/DS918+). DNS is here because I use Consul with dnsmasq. Some smart stuff lives here like my [ecobee](https://www.ecobee.com/).
* IoT Network: the real meat. This is nearly the same as the black hole one except it  can talk to the Server Network for C&C (esphome)


## The Software ##

First I played with [tasmota](https://tasmota.github.io) and the devices would randomly require me to set them up again.

Have you ever bought a device that turns on, sets up an AP for you to connect to then configure it? Thats a lot like Tasmota. 

The problem is that in just a few days I had to set up the same devices multiple times. So I'd go to do something, it wouldnt exist in homeassistant, and Id see a few `tasmota-#####` AP's I had to set up again. 

I solved this by pulling the source code down, modify the configs (which took a lot of figuring out for the settings), compile it, flash it. Per device. 

Then I had to upload the actual config for pins/buttons/etc. I hated this. Every second of it.

After playing around with **way** too many things, I finally landed on [esphome](https://esphome.io/).

Esphome is amazing. It has over-the-air updates, it automatically compiles based on some yaml and uploads it!
If it cant upload it (like the initial flash) I would simply set up the yaml, compile it, SCP it locally and flash it with [https://github.com/espressif/esptool](https://github.com/espressif/esptool). If this worked, it would now support OTA updates.

Because its a bunch of yaml I can source control it too!

It uses passwords for both API integrations (home-assistant) and for OTA updates (same or different password) which makes me feel good.

Also: the server runs as a container, so its running on my Server network on nomad. I can simply go to http://esphome.service.consul and click "Upload" to update my IoT.

{% img /images/esphome.png %}

As for the controlling the devices I chose home-assistant. It has great support for ESPHome.

{% img /images/hass.png %}

## The Devices ##

This is what you're really here for. Everything is ESP based for esphome.

I flashed almost all of these with an [ftdi serial adapter](https://www.amazon.com/HiLetgo-FT232RL-Converter-Adapter-Breakout/dp/B00IJXZQ7C) and [esptool.py](https://github.com/espressif/esptool) similar to 

```
#!/bin/bash
firmware="something.bin"
port="/dev/cu.usbserial-00000000"
esptool.py --port ${port} --baud 115200 erase_flash
esptool.py --port ${port} --baud 115200 write_flash 0x0 ${firmware}
```

### Garage Door Opener ###

This one took some learning to get right.
Its a [nodemcu ESP8266](https://www.amazon.com/HiLetgo-Internet-Development-Wireless-Micropython/dp/B081CSJV2V) hooked up to a 12v relay shimmed to the garage door trigger. Triggering GPIO16 triggers the relay.
Attached to GPIO5 is a reed switch on the garage door track that is open or closed. The reed switch is a magnetic based switch. When a magnet is near it, it  breaks the switch (open) meaning the garage door is closed. I wired it this way so that the only way the door can be considered "closed" is that the magnet has broken the sensor. No power, messed up sensor, manually opened (without using the garage door chain) etc would all be considered an open door. I chose this because I'd rather be safe than sorry and the door being closed is not an assumption but a fact.

{% img /images/esp-garage1.jpg %}
{% img /images/esp-garage2.jpg %}
{% img /images/esp-garage3.jpg %}
{% img /images/esp-garage4.jpg %}

```
esphome:
  name: garage_switch
  platform: ESP8266
  board: esp01_1m
wifi:
  ssid: !secret wifi_name
  password: !secret wifi_pass
  domain: !secret wifi_domain
logger:
api:
  password: !secret api_password
ota:
  password: !secret ota_password
binary_sensor:
  - platform: status
    name: "garage"
  - platform: gpio
    pin:
      number: 16
      inverted: False
      mode: INPUT_PULLUP
    name: "door"
    device_class: door
    filters:
      - delayed_off: 10ms
      - delayed_on: 10ms
switch:
  - platform: gpio
    name: "garage"
    pin: 5
```

### Sprinklers ###

This one is in progress. It works but I haven't finished the housing (needs to be gasket based and ABS).

This is based on a [wemos d1 mini](https://www.amazon.com/HiLetgo-Development-ESP8285-Wireless-Internet/dp/B07BK435ZW) and is essentially 2 [12v solenoid valves](https://www.amazon.com/gp/product/B010LT2GPG) on GPIO16 and GPIO14. the 12V is controlled based on circuitry through diodes and TIP120 fed from a 12V source (dropped to 5v via a LM7805 to the d1 mini).

{% img /images/solenoid_circuit.png %}
{% img /images/esp-sprinkler1.jpg %}
{% img /images/esp-sprinkler2.jpg %}

```
esphome:
  name: sprinklers
  platform: ESP8266
  board: esp01_1m
wifi:
  ssid: !secret wifi_name
  password: !secret wifi_pass
  domain: !secret wifi_domain
logger:
api:
  password: !secret api_password
ota:
  password: !secret ota_password
switch:
  - platform: gpio
    name: "sprinkler1"
    pin: 16
    inverted: no
  - platform: gpio
    name: "sprinkler2"
    pin: 14
    inverted: no
```

### Camera ###

This one is still a bit in progress. My end goal is to take snapshots on an interval and push them to s3 so that I can play around with machine learning, but right now it lets me view it in hass (no recording as of now).

I chose the [esp32-cam](https://www.amazon.com/gp/product/B07RXPHYNM) because it's very cheap. That's all, and it supports esphome.
The housing I 3d printed [from here](https://www.thingiverse.com/thing:3652452)

Note: This was a bit annoying to set up the first time. You can't "just flash it", you need to set up [spiffs](https://github.com/pellepl/spiffs) correctly. Which looked like this:

```
firmware="something.bin"
port="/dev/cu.usbserial-00000000"
# minimal spiffs:
esptool.py --port ${port} --baud 921600 --chip esp32 erase_flash
esptool.py --chip esp32 --port ${port} --baud 921600 --before default_reset --after hard_reset write_flash -z --flash_mode dio --flash_freq 80m --flash_size detect 0x1000 /Users/myoung/Library/Arduino15/packages/esp32/hardware/esp32/1.0.4/tools/sdk/bin/bootloader_qio_80m.bin 0x10000 ${firmware}
```

{% img /images/esp32-cam.jpg %}

```
esphome:
  name: frontdoor_cam
  platform: ESP32
  board: esp32dev
wifi:
  ssid: !secret wifi_name
  password: !secret wifi_pass
  domain: !secret wifi_domain
logger:
api:
  password: !secret api_password
ota:
  password: !secret ota_password
esp32_camera:
  name: frontdoor_cam
  external_clock:
    pin: GPIO0
    frequency: 20MHz
  i2c_pins:
    sda: GPIO26
    scl: GPIO27
  data_pins: [GPIO5, GPIO18, GPIO19, GPIO21, GPIO36, GPIO39, GPIO34, GPIO35]
  vsync_pin: GPIO25
  href_pin: GPIO23
  pixel_clock_pin: GPIO22
  power_down_pin: GPIO32
  resolution: 1600x1200
  max_framerate: 30 fps
  jpeg_quality: 25
```

### LED Strips ###

In the kids room I have some [star lights](https://www.amazon.com/Twinkle-Star-Waterproof-Extendable-Decoration/dp/B07FSLWPRB), and on our Pergola I have some [waterproof LED lights](https://www.amazon.com/Flexible-Waterproof-Christmas-Kitchen-Daylight/dp/B00HSF66JO) all controlled with:

[This magic home smart controller](https://www.amazon.com/Nexlux-Wireless-Controller-Compatible-Included/dp/B07116SX41) . I chose this because its ESP8266 based and flashable!


{% img /images/magichome.jpg %}

Wiring that up to the FTDI I flashed this yaml:

```
esphome:
  name: liam_room_starlights
  platform: ESP8266
  board: esp01_1m
wifi:
  ssid: !secret wifi_name
  password: !secret wifi_pass
  domain: !secret wifi_domain
logger:
api:
  password: !secret api_password
ota:
  password: !secret ota_password
light:
  - platform: rgbww
    name: "Liam Starlight"
    red: output_component1
    green: output_component2
    blue: output_component3
    cold_white: output_component4
    cold_white_color_temperature: 6536 K
    warm_white: output_component5
    warm_white_color_temperature: 2000 K

output:
  - platform: esp8266_pwm
    id: output_component1
    pin: 5

  - platform: esp8266_pwm
    id: output_component2
    pin: 12

  - platform: esp8266_pwm
    id: output_component3
    pin: 13

  - platform: esp8266_pwm
    id: output_component4
    pin: 15

  - platform: esp8266_pwm
    id: output_component5
    pin: 16
```

### Light Bulbs ###

For the bulbs I chose [globe brand](https://www.amazon.com/gp/product/B07RZ8QMG3) because theyre also ESP8266 based. 

For 2 of 3 of these I had success with [tuya-convert](https://github.com/ct-Open-Source/tuya-convert) which was pleasant.
Basically I built my .bin from the yaml below, replaced tasmota.bin in that dir with it because I'm sneaky, ran tuya-convert, flipped the light off/on 3 times and it flashed. 

For 2 of them (1 I broke), I had to get down and dirty. I basically tore it apart and followed [this guide](https://github.com/arendst/Tasmota/wiki/Mirabella-Genio-Bulb). Temporarily soldering and removing all the heat safe gunk is not fun. But 1 flashed. 1....caught fire....

{% img /images/bulb-fire.jpg %}

```
esphome:
  name: driveway_light
  platform: ESP8266
  board: esp01_1m
wifi:
  ssid: !secret wifi_name
  password: !secret wifi_pass
  domain: !secret wifi_domain
logger:
api:
  password: !secret api_password
ota:
  password: !secret ota_password
output:
  - platform: esp8266_pwm
    id: output_green
    pin: GPIO12
    max_power: 0.7
  - platform: esp8266_pwm
    id: output_red
    pin: GPIO4
    max_power: 0.7
  - platform: esp8266_pwm
    id: output_blue
    pin: GPIO14
    max_power: 0.7
  - platform: esp8266_pwm
    id: output_wwhite
    pin: GPIO13
    max_power: 0.65
  - platform: esp8266_pwm
    id: output_cwhite
    pin: GPIO5
    max_power: 0.65
light:
  - platform: rgbww
    name: "driveway_light"
    id: light_rgb
    red: output_red
    green: output_green
    blue: output_blue
    cold_white: output_cwhite
    warm_white: output_wwhite
    cold_white_color_temperature: 5000 K
    warm_white_color_temperature: 2000 K
    effects:
      - strobe:
      - flicker:
      - random:
    restore_mode: ALWAYS_ON
```

### Smart Plugs ###

These come in handy for seasonal stuff like xmas lights, etc. But especially for my 3d printer as an emergency safe-measure (forget to turn off etc)

I chose the [Sonoff S31](https://www.amazon.com/Sonoff-Compatible-Assistant-Supporting-Required/dp/B07YXVWC3Y). 

{% img /images/s31_1.jpg %}
{% img /images/s31_2.jpg %}

Easy to take apart, a bit annoying to flash (involves temporarily soldering), but work amazingly.

The yaml makes sure that its on by default and that the power button is a hard toggle for the power for manual control. Blue light indicates wifi, red indicates on/off for the relay.

```
esphome:
  name: plug1
  platform: ESP8266
  board: esp01_1m
wifi:
  ssid: !secret wifi_name
  password: !secret wifi_pass
  domain: !secret wifi_domain
logger:
api:
  password: !secret api_password
ota:
  password: !secret ota_password
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO0
      mode: INPUT_PULLUP
      inverted: True
    name: "Button"
    on_press:
      - switch.toggle: fakebutton
  - platform: template
    name: "Running"
    filters:
      - delayed_off: 15s
    lambda: |-
      if (isnan(id(power).state)) {
        return {};
      } else if (id(power).state > 4) {
        // Running
        return true;
      } else {
        // Not running
        return false;
      }
switch:
  - platform: template
    name: "POW Relay"
    optimistic: true
    id: fakebutton
    turn_on_action:
    - switch.turn_on: relay
    - light.turn_on: led
    turn_off_action:
    - switch.turn_off: relay
    - light.turn_off: led
  - platform: gpio
    id: relay
    pin: GPIO12
    restore_mode: ALWAYS_ON
output:
  - platform: esp8266_pwm
    id: pow_blue_led
    pin:
      number: GPIO13
      inverted: True

light:
  - platform: monochromatic
    name: "Blue LED"
    output: pow_blue_led
    id: led
    restore_mode: ALWAYS_ON

sensor:
  - platform: wifi_signal
    name: "WiFi Signal"
    update_interval: 60s
  - platform: uptime
    name: "Uptime"
  - platform: cse7766
    update_interval: 2s
    current:
      name: "Current"
    voltage:
      name: "Voltage"
    power:
      name: "POW Power"
      id: power
      on_value_range:
        - above: 4.0
          then:
            - light.turn_on: led
        - below: 3.0
          then:
            - light.turn_off: led
text_sensor:
  - platform: version
    name: "ESPHome Version"
```

### Smart Switch ###

I kept blowing my outdoor smart bulb, I'm guessing the voltage or current is too volatile?

I chose the [TreatLife single pole](https://www.amazon.com/gp/product/B07R7PCCT9). 

{% img /images/ss01s.png %}

This yaml lets the LED indicator (GPIO4) toggle with the virtual button (GPIO13) and the physical button (GPIO12)

```
esphome:
  name: front_porch_switch
  platform: ESP8266
  board: esp01_1m

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  domain: !secret wifi_domain

logger:
  level: VERBOSE
  baud_rate: 0

api:
  password: !secret api_password

ota:
  password: !secret ota_password

switch:
  - platform: gpio
    id: "relay"
    name: "light"
    pin: 12

output:
  - platform: esp8266_pwm
    id: status_led
    pin:
      number: GPIO4
      inverted: True

binary_sensor:
  - platform: gpio
    name: "button"
    pin:
      number: 13
      inverted: True
    on_press:
      - switch.toggle: relay
```
