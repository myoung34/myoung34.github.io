---
title: 'Raspberry Pi-Automated Bluetooth Audio Player !'
author: myoung
layout: post
comments: true
permalink: /post/raspberry-pi-automated-bluetooth-audio-player/
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - Linux
  - Media
  - Programming
---
<!--more-->
## Backstory

After a long break, I&#8217;ve finally gotten one part of my to-do list for my Raspberry Pi&#8230;.automated bluetooth audio playing!  
As always: there were issues. First, the support for this sucked. I was thankful to get a starting point from <a title="John Hamelink" href="https://github.com/johnhamelink/bluetoothradio" target="_blank">John Hamelink</a>. He wrote a set of sweet scripts that automate a lot of this. However, they were written for Arch Linux, and required a bit of work syntactically to get it working for Debians Wheezy Release. <a href="https://github.com/myoung34/bluetoothradio" target="_blank">My final code is here.</a>

## The Nitty Gritty (install steps)

First, install the normal packages (as seen here) and let the pi user able to control bluetooth devices: 

```
pi@raspberrypi ~ $ sudo apt-get install bluez pulseaudio-module-bluetooth python-gobject python-gobject-2
pi@raspberrypi ~ $ sudo usermod -a -G lp pi
```

Next, let&#8217;s make the bluetooth module compatible for audio streaming. Edit **/etc/bluetooth/audio.conf** and add this after [General]: 

```
Enable=Source,Sink,Media,Socket
```


Next, we&#8217;ll let pulseaudio do the playback. Edit **/etc/pulse/daemon.conf** and uncomment: 

```
resample-method = trivial ; ADD THIS LINE TO THE FILE!
```


We need some additional packages to make it compatible with my scripts. Qdbus allows you to send D-Bus messages to the bluez daemon, git-core is so you can clone my repo, and bluez-tools provides the bluetooth-agent which is one of the daemon pieces. 

```
pi@raspberrypi ~ $ sudo apt-get install bluez-tools qdbus git-core
```


Now, we&#8217;re going to gain root, put my code in /root, and make a init.d file you can provide at startup. 

```
pi@raspberrypi ~ $ sudo su -
root@raspberrypi:~# git clone https://github.com/myoung34/bluetoothradio.git --branch Wheezy
root@raspberrypi:~# cd bluetoothradio
root@raspberrypi:~/bluetoothradio# cp bluetooth-server /etc/init.d
root@raspberrypi:~/bluetoothradio# chmod 755 /etc/init.d/bluetooth-server && chmod +x /etc/init.d/bluetooth-server
root@raspberrypi:~/bluetoothradio# update-rc.d bluetooth-server defaults
root@raspberrypi:~/bluetoothradio# reboot
```



## Usage

After the reboot, you can now pair your device using &#8217;1234&#8242; (unless you modified the bluetoothPin file), and play yo&#8217; music!

If for some reason it&#8217;s not connection, visit <a title="KMonkey711" href="https://kmonkey711.blogspot.com/" target="_blank">KMonkey711&#8242;s tutorial</a> and follow the steps to connect and trust the device, and reboot the pi.

## Acknowledgements

Thanks to:  
<a title="johnhamelink" href="https://github.com/johnhamelink" target="_blank">johnhamelink</a>  
<a title="KMonkey711" href="https://kmonkey711.blogspot.com/" target="_blank">KMonkey711</a>
