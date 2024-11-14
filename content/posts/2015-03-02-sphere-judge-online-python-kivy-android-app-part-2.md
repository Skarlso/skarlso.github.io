+++
author = "hannibal"
categories = ["android", "programming", "Python"]
date = "2015-03-02"
title = "Sphere Judge Online – Python Kivy Android app – Part 2"
url = "/2015/03/02/sphere-judge-online-python-kivy-android-app-part-2/"

+++

Here we are again. I will attempt to further this little journey of mine into the land of Android and Python.

This is the second part of the advanture you can read the first one a little bit back.

# The Script

We left off at a point where I successfully configured my environment and compiled my first hello world APK. At that point it took a little bit fiddling to get it to work on my phone.

Now, I have progressed a little bit into spoj's page parsing. The code so far is as follows:

~~~python

__author__ = 'hannibal'

import requests
from random import randint
import lxml.html as lh

random_page_number = randint(, 63) # 63 being the maximum page number at spoj
r = requests.get("http://www.spoj.com/problems/classical/sort=0,start=%d" % (random_page_number * 50 - 50))
# Problem Div XPath => //[@class="problems"]

spoj_page = lh.document_fromstring(r.text)
links_to_problems = spoj_page[].xpath("//tr[@class='problemrow']/td[2]/a")

current_link = links_to_problems[randint(, len(links_to_problems))]
print(current_link.attrib['href'])
r = requests.get("http://www.spoj.com/%s" % current_link.attrib['href'])

print(r.text.encode("utf-8"))
~~~

This is pretty straight forward so far. It gets the problems page, loads in all of the links and prints it out.

My goal is an application which looks something like this:

\___\___\___\___\___\___\___\___\___\___\___\___\___\___\___

|   \___\___\___\___\___\___\___\___\___\___\___\___\___\___  |

|  |                                                                           | |

|  |                                                                           | |

|  |                    Display Problem Description               | |

|  |                                                                           | |

|  |                                                                           | |

|  |                                                                           | |

|  |\___\___\___\___\___\___\___\___\___\___\___\___\_____ | |

|                                                                                 |

|                                                                                 |

|                         Button:Finish Problem                        |

|                                                                                 |

|                         Button:Next Problem                          |

|\___\___\___\___\___\___\___\___\___\___\___\___\___\_____ |

It's very basic. When it loads up, it will gather and display a new problem. You have two options, either get a new one, or save / finish this item, saying you never want to see it again.

Let's put the first part into an android app. Just gather data, and get it disaplyed.

\*Queue a days worth of hacking and frustrated cussing.\*

So, turns out it's not as easy as I would have liked it to be. I ran into some pretty nasty problems. Some of them I'll write down below for the record, and an attempted solution as well.

# Problems

**#1:** **Problem:** Libraries. I'm using lxml and requests. Requests is a pure python library, but lxml is partially C. Which apparently is not very well supported yet.

**Solution (Partial):** I could optain request by two ways, but the most simple one, was basically just building my distribution with the optional requests module like this:

~~~

./distribute.sh -m "openssl pil requests kivy"
~~~

Attempting to do the same with LXML resulted in a compile issue which I tracked down to something like: "sorry, but we don't support OSX". But it's okay. There are other ways to parse an html page, I just really like the xpath filter. So I soldiered on with trying to get something to work at least.

**#3: Problem:** _Bogus compile time exception._ There were some exceptions on the way when I was trying to compile with buildozer. **Solution:** It's interesting because previously my solution to another compile time issue was to use a specific version of Cython. But this time the solution was to actually remove that version and install the latest one. Which is 0.22 as of the time of this writing. So:

~~~

sudo pip update cython
~~~

**#2: Problem:** Connection. So now, I'm down to the bare bone. At this point, I just want to see a page source in a label. My code looks like this:

~~~python

import kivy
kivy.require('1.8.0')
from kivy.lang import Builder
from kivy.uix.gridlayout import GridLayout
from kivy.properties import NumericProperty
from kivy.app import App

import requests
from random import randint
# import lxml.html as lh

# import sys
# sys.path.append('/sdcard/com.googlecode.pythonforandroid/extras/python/site-packages')


Builder.load_string('''
:
    cols: 1
    Label:
        text: root.get_problem()
    Button:
        text: 'Click me! %d' % root.counter
        on_release: root.my_callback()
''')

class SpojAppScreen(GridLayout):
    counter = NumericProperty()
    def my_callback(self):
        print 'The button has been pushed'
        self.counter += 1

    def get_problem(self):
        random_page_number = randint(, 63) # 63 being the maximum page number at spoj
        r = requests.get("http://www.spoj.com/problems/classical/sort=0,start=%d" % (random_page_number * 50 - 50))

        # Problem Div XPath => //[@class="problems"]

        # spoj_page = lh.document_fromstring(r.text)
        # links_to_problems = spoj_page[0].xpath("//tr[@class='problemrow']/td[2]/a")

        # current_link = links_to_problems[randint(0, len(links_to_problems))]
        # print(current_link.attrib['href'])
        # r = requests.get("http://www.spoj.com/%s" % current_link.attrib['href'])
        # print(r.text.encode("utf-8"))
        return r.text.encode("utf-8")

class SpojApp(App):
    def build(self):
        return SpojAppScreen()

if __name__ == '__main__':
    SpojApp().run()
~~~

However, running this results in a connection error in adb logcat:

~~~

I/python  (27610):  kivy.lang.BuilderException: Parser: File "", line 5:
I/python  (27610):  ...
I/python  (27610):        3:    cols: 1
I/python  (27610):        4:    Label:
I/python  (27610):  &gt;&gt;    5:        text: root.get_problem()
I/python  (27610):        6:    Button:
I/python  (27610):        7:        text: 'Click me! %d' % root.counter
I/python  (27610):  ...
I/python  (27610):  BuilderException: Parser: File "", line 5:
I/python  (27610):  ...
I/python  (27610):        3:    cols: 1
I/python  (27610):        4:    Label:
I/python  (27610):  &gt;&gt;    5:        text: root.get_problem()
I/python  (27610):        6:    Button:
I/python  (27610):        7:        text: 'Click me! %d' % root.counter
I/python  (27610):  ...
I/python  (27610):  ConnectionError: ('Connection aborted.', gaierror(4, 'non-recoverable failure in name resolution.'))
~~~

**Solution:** I tried simply putting out a random number at some point, which actullay worked, so I know it's the connection. I'm guessing I need permission to access the network. Which would be this:

~~~

uses-permission android:name="android.permission.INTERNET"
~~~

And yes! Building and installing it with this additional permission got me so far as I can display the web page's content in a label.

~~~

./build.py --package org.spoj --permission INTERNET --name "Spoj" --version 1.0 --dir /Users/hannibal/PythonProjects/spoj/ debug
~~~

There is a saying that you should end on a high note, so that is what I'm going to do here right now. Join me next time, when I'll try to replace lxml with something else.

Thanks for reading!

Gergely.