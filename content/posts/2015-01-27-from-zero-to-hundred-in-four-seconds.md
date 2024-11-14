+++
author = "hannibal"
categories = ["scala"]
date = "2015-01-27"
title = "From Zero to Hundred in Four seconds"
url = "/2015/01/27/from-zero-to-hundred-in-four-seconds/"

+++

I thought I throw my grudge out of the window against Scala and try something with it.

I also got my hands on a fairly new book, called: "<a href="http://www.amazon.co.uk/Learning-Scala-Practical-Functional-Programming/dp/1449367933/ref=sr_1_1?ie=UTF8&qid=1422340157&sr=8-1&keywords=learning+scala+a+practical" target="_blank">Learning Scala: Practical Functional Programming for the JVM</a>". Turns out to be a rather fun book to read. And Jason Swartz has a nice way of writing. So I wanted to play around with <a href="https://www.playframework.com/" target="_blank">Play 2 Framework</a>. It now comes packaged in <a href="https://typesafe.com/get-started" target="_blank">Activator</a>.

So, I started the long path from almost zero to handle all that. I'm running the latest Ubuntu ( 14 ) and latest Java ( 8 ). The list: Scala, SBT, IntelliJ, Play ( through activator ).

I was pleased that, considering a network which allowed me a download speed of ~1.5MB/s ( that's byte, not bit ), I was up and running in about 4 minutes. That's Play running, with a test application created through activator and then imported into an IntelliJ Scala project.

I'm impressed.

I added SBT through the package manager like this:

    echo "deb http://dl.bintray.com/sbt/debian /" | sudo tee -a /etc/apt/sources.list.d/sbt.list
    sudo apt-get update
    sudo apt-get install sbt

It's really simple.

After that, I did an apt-get on the latest Scala.

I already had IntelliJ.

Activator download took me ~1 minute; then I executed the command to create a test app:

<pre class="prettyprint prettyprinted"><code class="language-bash">&lt;span class="pln">$ activator new my&lt;/span>&lt;span class="pun">-&lt;/span>&lt;span class="pln">first&lt;/span>&lt;span class="pun">-&lt;/span>&lt;span class="pln">app play&lt;/span>&lt;span class="pun">-&lt;/span>&lt;span class="pln">scala&lt;/span></code></pre>

Simple. Activator downloaded everything my system was still missing.

Then run the start command:

<pre class="prettyprint prettyprinted"><code class="language-bash">&lt;span class="pln">$ cd my&lt;/span>&lt;span class="pun">-&lt;/span>&lt;span class="pln">first&lt;/span>&lt;span class="pun">-&lt;/span>&lt;span class="pln">app
$ activator&lt;/span></code></pre>

And you are ready to rock & roll. Importing it in IntelliJ was a blink of an eye.

I'm really impressed with how easy getting started became with these projects and frameworks. I remember a time where I had to configure everything, get tomcat and the whole JVM or Jetty or whatnot, and try to get up and running took half a day at least. Would my internet be faster, I think this would have been even less.

I'll post more as I go forward.

As always,

Thanks for reading.