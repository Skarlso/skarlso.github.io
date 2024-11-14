+++
author = "hannibal"
categories = ["programming", "Python"]
date = "2015-03-15"
tags = ["mathematics"]
title = "Python and my Math commitment"
url = "/2015/03/15/python-and-my-math-commitment/"

+++

Let's talk about plans. It's good to have one. For example, I have a plan for this year.

I kind of like math. So, I have this book:

It's 1400 pages long and basically, has everything in it. It's a rather exhaustive book. Hence, my plan is to finish the book by the end of 2015 and write a couple of python scripts that calculate something interesting.

(2021 Hindsight): Yeah, I didn't manage this... But it's a cool idea, let's see if I can get around coming further. I managed to get until 500 pages or so, before life stepped in.

For example, Newton's law of cooling how I learned it is:

~~~
k*log(2.5)*((t(0)-k)/(t-k))
~~~

Where k => a material's surface based constant. T(0) => initial temperature. T => target temperature. K => Environment's temperature.

A simple python script for this:

~~~python

# Calculating Newton's law of Cooling
from __future__ import division
import sys
from math import log

def calculation(k, Tz, T, K):
	res = (Tz - K)/(T - K)
	return k * (log(res, 2.5))

k = sys.argv[1]
Tz = sys.argv[2]
T = sys.argv[3]
K = sys.argv[4]

print("Calculating aproximate temperature for given parameters: k=%s, Tz=%sC, T=%sC, K=%sC" % (k, Tz, T, K))

print(calculation(float(k), float(Tz), float(T), float(K)))
~~~

Enjoy.

And as always,
Thanks for reading!
