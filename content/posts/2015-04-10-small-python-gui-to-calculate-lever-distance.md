+++
author = "hannibal"
categories = ["programming", "Python"]
date = "2015-04-10"
tags = ["mathematics"]
title = "Small Python GUI to Calculate Lever Distance"
url = "/2015/04/10/small-python-gui-to-calculate-lever-distance/"

+++

Hi folks.

Just a small script which calculates your distance from a lever focal point if you know your weight, the object's weight and the object's and the distance the object has from the focal point of the lever.

Like this:

This script will give you D1. And this is how it will look like in doing so:

So, in order for me (77kg) to lift an object of 80kg which is on a, by default, 1 meter long lever, I have to stand back ~1.03meters. Which is totally cool, right?

Here is the code:

~~~python

from Tkinter import *
import ttk

def calculate(*args):
    try:
        your_weight_value = float(your_weight.get())
        object_weight_value = float(object_weight.get())
        object_distance_value = float(object_distance.get())
        your_distance.set((object_weight_value * object_distance_value) / your_weight_value)
    except ValueError:
        pass

root = Tk()
root.title("Lever distance counter")

mainframe = ttk.Frame(root, padding="4 4 12 12")
mainframe.grid(column=, row=, sticky=(N, W, E, S))
mainframe.columnconfigure(, weight=1)
mainframe.rowconfigure(, weight=1)

your_weight = StringVar()
object_weight = StringVar()
object_distance = StringVar()
your_distance = StringVar()

object_distance.set("1")

your_weight_entry = ttk.Entry(mainframe, width=7, textvariable=your_weight)
your_weight_entry.grid(column=2, row=1, sticky=(W, E))
object_weight_entry = ttk.Entry(mainframe, width=7, textvariable=object_weight)
object_weight_entry.grid(column=2, row=2, sticky=(W, E))
object_distance_entry = ttk.Entry(mainframe, width=7, textvariable=object_distance)
object_distance_entry.grid(column=2, row=4, sticky=(W, E))


ttk.Label(mainframe, textvariable=your_distance).grid(column=2, row=3, sticky=(W, E))
ttk.Label(mainframe, text="Your weight").grid(column=1, row=1, sticky=W)
ttk.Label(mainframe, text="Object weight").grid(column=1, row=2, sticky=W)
ttk.Label(mainframe, text="Object Distance").grid(column=1, row=3, sticky=W)
ttk.Label(mainframe, text="Your Distance").grid(column=1, row=4, sticky=W)

ttk.Label(mainframe, text="kg").grid(column=3, row=1, sticky=W)
ttk.Label(mainframe, text="kg").grid(column=3, row=2, sticky=W)
ttk.Label(mainframe, text="m").grid(column=3, row=3, sticky=W)
ttk.Label(mainframe, text="m").grid(column=3, row=4, sticky=W)

ttk.Button(mainframe, text="Calculate", command=calculate).grid(column=3, row=5, sticky=W)

for child in mainframe.winfo_children(): child.grid_configure(padx=5, pady=5)

your_weight_entry.focus()
root.bind('', calculate)

root.mainloop()
~~~

Please enjoy, and feel free to alter in any way. I'm using Tkinter and a grid layout which I find very easy to work with.

Thanks for reading,
Gergely.
