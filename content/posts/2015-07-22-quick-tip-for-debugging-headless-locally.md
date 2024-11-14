+++
author = "hannibal"
categories = ["problem solving", "rambling"]
date = "2015-07-22"
tags = ["Packer", "vagrant"]
title = "Quick Tip for Debugging Headless Locally"
url = "/2015/07/22/quick-tip-for-debugging-headless-locally/"

+++

If you are installing something with Packer and you have Headless enabled(and you are lazy and don't want to switch it off), it gets difficult, to see output.

Especially on a windows install the Answer File / Unattended install can be like => Waiting for SSH. for about an hour or two! If you are doing this locally fret not. Just start VirtualBox, and watch the Preview section which will display the current state even if it's a headless install!

It's a small windows, but your can click on **Show** which will open the VM in a proper view.

Enjoy,
Gergely.
