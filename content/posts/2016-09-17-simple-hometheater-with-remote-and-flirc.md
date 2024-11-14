+++
author = "hannibal"
categories = ["Hardware"]
date = "2016-09-17"
title = "Budget Home Theather with a Headless Raspberry Pi and Flirc for Remote Controlling"
url = "/2016/09/17/simple-hometheater-with-remote-and-flirc"

+++

Intro
------

Hello folks.

Today, I would like to tell you about my configuration for a low budget Home Theater setup.

My tools are as follows:

 * [FLIRC](https://flirc.tv/)
 * [Raspberry Pi 2](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/)
 * 500G SSD
 * An a good 'ol wifi

TL;DR
------

Use Flirc for remote control, `omxplayer` for streaming the movie from an SSD on a headless PI controller via SSH and enjoy a nice, cold Lemon - Menta beer.

Flirc
------

First, the remote control. So, I like to sit in my couch and watch the movie from there. I hate getting up, or having a keyboard at arm length to control the pi. Flirc is a very easy way of doing just that with a simple remote control.

It costs ~$22 and is easy to setup. Works with any kind of remote control. Setting up key bindings for the control, is as simple as starting the Flirc software and pressing buttons on the remote to map to keyboard keys. Now, my pi is running headless, and the Flirc binary isn't quite working with raspbian; so to do the binding, I just did that on my main machine. When I was done, I just plugged in the Flirc, and proceeded to setup the pi.

Raspberry Pi 2
----------------

The pi 2 is a small powerhouse. However, the SD card on which it sits is simply not fast enough. From time to time, I experienced lateness in sound, or stutter in video. So, instead of having the movie on the pi, I'm streaming through a faster SSD with [SSHFS](https://github.com/libfuse/sshfs). For playing, I'm using `omxplayer`. With omxplayer, I had a few problems, because sound was not coming through the HDMI cable. A little bit of research lead me to this change in the pi's boot config. Uncomment this line:

~~~bash
#hdmi_driver=2
~~~

After rebooting, I also, did this thing:

~~~bash
sudo apt-get install alsa-utils
sudo modprobe snd_bcm2835
sudo amixer -c 0 cset numid=3 2
~~~

This saved my bacon. The whole answer can be found here: [Stackoverflow](http://raspberrypi.stackexchange.com/questions/44/why-is-my-audio-sound-output-not-working).

Once SSHFS was working, and HDMI received sound, I just executed this command: `omxplayer -o hdmi /media/stream/my_movie.mkv`. This told omxplayer to use the local HDMI connection for video output.

All this was from my computer through an SSH session so I never controlled the pi directly. Once done, I proceeded to sit down with a nice, cold Lemon - Menta beer and a remote control.

Once little gotcha -- `omxplayer` is controlled through the buttons + (volume up), - (volume down), <SPACE> (stop, play), and q for quitting. Flirc is able to map any key *combinations* on a keyboard as well to any button on the remote. Combinations can be done by selecting a control key and pressing another key. So mapping `+` to the volume up button was by pressing shift and then '='.

Wrapping Up
------------

I enjoyed the movie while being able to adjust the volume, or pause it, when my popcorn was ready, and close the player when the movie was done. There are a number of other ways to do this, like using [kodi](https://kodi.tv/) + [yatse](https://play.google.com/store/apps/details?id=org.leetzone.android.yatsewidgetfree&hl=en). Which lets you remote control a media software with your mobile phone. But I'm using the pi for a number of other things and the GUI is rather resource heavy.

There you have it folks. Might not be the easiest setup, but it's pretty awesome anyways.

Cheers,
Gergely.
