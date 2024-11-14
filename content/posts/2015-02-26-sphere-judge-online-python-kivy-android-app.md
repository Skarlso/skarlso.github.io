+++
author = "hannibal"
categories = ["android", "programming", "Python"]
date = "2015-02-26"
title = "Sphere Judge Online – Python Kivy Android app"
url = "/2015/02/26/sphere-judge-online-python-kivy-android-app/"

+++

Hello folks.

Today I would like to take you on a journey I fought myself through in order to write a python android app, which gets you a random problem from <a href="http://www.spoj.com/problems/classical/" target="_blank">Sphere Judge Online</a>. Then you can mark it as solved and it will be stored as such, and you can move on to the next problem. With the words of Neil deGrasse Tyson, Come with Me!

# Beginnings

When I first embarked on this endeavour I ran into numerous errors, many amongst them being compilation issues when I was trying to install libraries.

I started to write down all of these, and then started fresh on a new machine. I realised that ALL of my problems where only because of **ONE **thing. One thing, which I wanted to do, but it ended up being the death of me. And that is.. \*Drummrolls\* **Python 3. **I tried doing all the things that I started to do, with Python 3. Turns out, that neither libraries are supporting it very well yet. And that's including Cython as well, which I thought would be up to speed by now. But sadly, it's not.

# Prerequisite

In order to go any further we need a few things first. For this to work, you'll have to perform these things in order as I found out later. And certain versions of certain libraries are required instead of the latest ones.

Depending on the environment you are using, you need to install python-dev and some other graphic libraries. I followed this and that was fine. Latest packages are working alright.

~~~bash
sudo apt-get install build-essential patch git-core ccache ant python-pip python-dev
sudo apt-get install ia32-libs  libc6-dev-i386
sudo apt-get install lib32stdc++6 lib32z1
~~~

Only install these if you are absolutely certain you need them.

Clone python-android from git into a nice and cosy directory.

~~~
git clone https://github.com/kivy/python-for-android.git
~~~

While this is underway, for python-android you also need <a href="http://developer.android.com/sdk/index.html#Other" target="_blank">android-sdk</a> and <a href="https://developer.android.com/tools/sdk/ndk/index.html" target="_blank">android-ndk</a>. Select the ones which are for your environment. The NDK is needed in order to build the APK out of our python code later on.

After you are done, run ./android and install tools, APIs and other things you want. Make sure you have these set up:

~~~
export ANDROIDSDK=/path/to/android-sdk
export ANDROIDNDK=/path/to/android-ndk
export ANDROIDNDKVER=rX
export ANDROIDAPI=X
export PATH=$ANDROIDNDK:$ANDROIDSDK/platform-tools:$ANDROIDSDK/tools:$PATH
~~~

The API version needs to be the one which you installed on your machine.

Now, we have to get a specific version of Cython. In order to do that, execute the following command:

~~~
sudo pip install -I https://pypi.python.org/packages/source/C/Cython/Cython-0.20.1.tar.gz
~~~

Source your new .bash_profile file if you haven't done so already.

At this point we are ready to install Kivy. Please follow the instructions for your environment on the respective page from Kivy's documentation:

<a href="http://kivy.org/docs/installation/installation.html" target="_blank">http://kivy.org/docs/installation/installation.html</a>

**Note**: For Mac users. In addition, before doing the kivy stuff, and if you would like to execute kivy applications on your mac, you need to install pygame.

It's a bit of a hassle but you only need to perform these commands:

Install Quartz => <a href="http://xquartz.macosforge.org/landing/" target="_blank">http://xquartz.macosforge.org/landing/</a>

Install Homebrew => <span style="color: #ff9900;">ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)"</span>

Install some other packages => <span style="color: #ff9900;">brew install hg sdl sdl_image sdl_mixer sdl_ttf portmidi</span>

Install pygame => <span style="color: #ff9900;">pip install hg+http://bitbucket.org/pygame/pygame</span>

Once this finishes, you should be good to go for the final command in the prerequisites. Go to your cloned python-android folder and run this (make sure you have ANT installed):

~~~
./distribute.sh -m "openssl pil kivy"
~~~

Now we are ready for some coding.

# Implementation

So, finally after our environment is all setup, we can move on to write some python code. Let's start with a simple hello world application:

~~~python
import kivy
kivy.require('1.8.0') # 1.8.0 is the latest kivy version
from kivy.lang import Builder
from kivy.uix.gridlayout import GridLayout
from kivy.properties import NumericProperty
from kivy.app import App

Builder.load_string('''
:
    cols: 1
    Label:
        text: 'Welcome to the Hello world'
    Button:
        text: 'Click me! %d' % root.counter
        on_release: root.my_callback()
''')

class HelloWorldScreen(GridLayout):
    counter = NumericProperty()
    def my_callback(self):
        print 'The button has been pushed'
        self.counter += 1

class HelloWorldApp(App):
    def build(self):
        return HelloWorldScreen()

if __name__ == '__main__':
    HelloWorldApp().run()
~~~

This is a simple Hello World python-android app. Save this into a file called <span style="color: #ff9900;">main.py</span>. Main.py is used to execute the app on your phone. It's your entry point. Whatever app you are writing, this has to be where it will begin.

In order to get this installed on our device, we will use python-android's distribution.sh. The command to run after you changed directory into python-android is this (make sure that you have a compatible android device plugged in and in developer mode):

~~~

./build.py --package org.hello.world --name "Hello world" --version 1.0 --dir /PATH/TO/helloworld debug installd
~~~

Upon success, you should see it on your device. This is how the hello world app looks like:

# Finishing up

This has been quite the ride so far. We will continue our journey when I'll start writing my own app for SPOJ.

Thanks for reading!
Gergely.
