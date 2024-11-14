+++
author = "hannibal"
categories = ["android", "JavaScript", "knowledge", "programming"]
date = "2015-01-29"
title = "JavaScript Web Framework – Meteor"
url = "/2015/01/29/javascript-web-framework-meteor/"

+++

Hi,

This time I would like to write about something that interests me. I wanted to try out a pure JavaScript web framework.

My choice is: <a href="https://www.meteor.com/" target="_blank">Meteor</a>. Looks interesting enough and it was recommended by a friend of mine. So, let's dive in.

#### **Installation**

As always, one starts with installation. The page tells us to follow this simple step:

~~~
curl https://install.meteor.com/ | sh
~~~

Easy enough, when you are on Linux. Turns out, that there is no official release yet for Windows. I'm in luck then. After running the command though, I saw this popping up into my face:

~~~
curl: (60) Peer certificate cannot be authenticated with known CA certificates
More details here: http://curl.haxx.se/docs/sslcerts.html
~~~

There is always something. in that case a more accurate command to use would be the following:

~~~
curl -k https://install.meteor.com/ | sh
~~~

This will force an insecure download. You might not face this issue, but just in case you do, use this command instead.

Branching off here. For those of you whom the curl didn't work because you are sitting behind a proxy you can specify a -proxy  protocol//username:password@proxy:port after your curl. Of course if that doesn't work then the script won't work either.

So open the script in one of your favourite editors, for me it's Sublime text, and find this line: "_Downloading Meteor distribution_". Lo, and behold; it uses curl. This is the only one in the script, so just edit it by adding in your -proxy setting as before and you should be right on track.

If that still gives you problems, try this:

Assuming that your browser is set up correctly with the proxy and just command line commands aren't working, you can go to this URL defined by the variable TARBALL_URL:

~~~
TARBALL_URL="https://d3sqy0vbqsdhku.cloudfront.net/packages-bootstrap/${RELEASE}/meteor-bootstrap-${PLATFORM}.tar.gz"
~~~

Note that there are two variables in there. For me these are:

RELEASE: 1.0.3.1

PLATFORM: os.linux.x86_64

The full URL is:

~~~
https://d3sqy0vbqsdhku.cloudfront.net/packages-bootstrap/1.0.3.1/meteor-bootstrap-os.linux.x86_64.tar.gz
~~~

Download the latest tarball and delete the CURL AND TAR command on the following line. After that, you just have to extract the tarball and move the directory to ~/.meteor.

Now you can run your sh again and you should be on the road, for sure this time.

Just to make sure, these are the line which you need to comment out:

~~~bash

# If you already have a tropohouse/warehouse, we do a clean install here:
if [ -e "$HOME/.meteor" ]; then
echo "Removing your existing Meteor installation."
rm -rf "$HOME/.meteor"
fi

TARBALL_URL="https://d3sqy0vbqsdhku.cloudfront.net/packages-bootstrap/${RELEASE}/meteor-bootstrap-${PLATFORM}.tar.gz"

INSTALL_TMPDIR="$HOME/.meteor-install-tmp"
rm -rf "$INSTALL_TMPDIR"
mkdir "$INSTALL_TMPDIR"
echo "Downloading Meteor distribution"
curl --proxy https://ggbrau:Daleks37@10.120.28.130:80--progress-bar --fail "$TARBALL_URL" | tar -xzf - -C "$INSTALL_TMPDIR" -o
# bomb out if it didn't work, eg no net
test -x "${INSTALL_TMPDIR}/.meteor/meteor"
mv "${INSTALL_TMPDIR}/.meteor" "$HOME"
rm -rf "${INSTALL_TMPDIR}"
# just double-checking :)
test -x "$HOME/.meteor/meteor"
~~~

#### Getting started

After a nice installation process we can continue to the getting started phase.

So, the documentation tells us that we have to simply execute a command.

~~~
meteor create simple-todos
~~~

At this point we should get a directory structure which is written in the manual. And, behold, that's exactly what happened. As usually, creating a skeleton is easy. Lets run the app. For that, the command is:

~~~
meteor
~~~

I can do that, I think.

And sure enough, I've got this little message, which I actually expected to see:

~~~
Can't listen on port 3000. Perhaps another Meteor is running?
~~~

In this world, where there are tons of applications running on your dev environment at any given time, it's possible to have something already running on the port 3000. Luckily this is something that's anticipated by now, and we are presented with an option to add in a proxy setting of our choice with -port <port>.

After I did that, I've got a nice confirm message that meteor is up and running. A quick check on the presented URL provided me with the confidence that my app is indeed reachable.

~~~
App running at: http://localhost:9999/
~~~

#### After Getting Started.

Now that we know that it's up and running we can continue with the tutorial. Up comes next a simple Todo list application with Templates. It's telling us to replace the code in the default starter app. At this point I'm wondering if it can hotswap. It should, since javascript and HTML is dynamic so there should be no problems there, right?

And sure enough, the moment I replaced the code and checked on my server status, I could see this:

~~~
Client modified -- refreshing
Meteor server restarted
~~~

With a brief flash of "Rebuilding.". So it does sort of work. It did, however, restart the server it just did it without your manual intervention. Which is nice, but on a larger scale application it might prove to be a tad bit annoying. For example, I add another item to the list, and suddenly, the server is restarted.

Since, I am a tester, let's see how it handles some problems.

I modified the JavaScript so that it has a syntax error.

~~~javascript

// simple-todos.js
if (Meteor.isClient) {
  // This code only runs on the client
  Template.body.helpers({
    tasks: [
      { text: "This is task 1" },
      { text: "This is task 2" },
      { text: "This is task 3" }
      { text: "This is task 4" }
    ]
  });
}
~~~

Note the missing ",". And, nicely enough I'm getting an error message telling me that I messed something up:

~~~
Errors prevented startup:

While building the application:
my_cool_app.js:10:7: Unexpected token {

Your application has errors. Waiting for file change.
~~~

It even tells you where the error is and it's waiting for you to fix it. After I've corrected my error it compiled fine and the application is up and running. Deleting the files did little difference as did corrupting the HTML pages or the CSS file. Nothing to see here, moving on.

#### Android Device

I'm sure everybody can read a manual and continue with collections, forms, events and such. What I'm more interested in is that Meteor promises it can run on Android devices. Now that perked my curiosity. With the rise of mobile devices, the desktop platform is slowly pushed back into a dark corner where even a <a href="http://mistborn.wikia.com/wiki/Tineye" target="_blank">Tineye </a>would have problems seeing it.

Hence, I want to see how easy it really is.

Meteor gives you a set of commands to install the android sdk and droid support for your application, which is nice. You just need to run this:

~~~

meteor install-sdk android
meteor add-platform android # Perform this step in the app's folder and agree to terms and conditions.
~~~

Now, if you are like me, someone who has experience with the android SDK and its emulator, you'll know that running that thing requires more time and processing power than simulating the chances of Leonardo DiCaprio winning an Oscar. I'll use a real device instead. For that, it appears I only have to run a simple command again.

~~~
meteor run android-device
~~~

And sure enough the app appeared on my device.

This is actually quite awesome. I only plugged in my device, enabled developer options and USB debugging and that's it. I'm quite impressed so far with Meteor and the Power of JavaScript. The app is on my phone and the static JavaScript parts are still working even though I shut the server down.

So my next burning question is. Will it Blend? I mean, Perform?

#### Benchmarking

So, now that I know that using, installing and getting started is pretty simple, what I also would like to know is how well it performs.

I have a quad core i7 16GB RAM Samsung SSD running Linux. Let's see 100 threads 10 second interval 10 times loop for a start. Look at how gorgeous this is.

40ms on average. Now let's crank it up and I'm performing the test on a separate machine but still on the same network. 1000 threads.

This time I've got a bit more churn and my pc started to fan like there is no tomorrow. But the server stayed stable. Latency did not waver for a bit. Next, 10.000 for as long as my machine can handle it.. Better save my work. Hah, my JMeter died. But it clocked at an average of 1000ms response time and the server stayed absolutely stable with no package lost, or errors.

#### Conclusion

I can say with a full heart that I'm impressed by Meteor and I very much like it. It's easy to use, even more easy to install and definitely can handle itself given that it's rather lightweight. The hot swapping / server re-starting can't be avoided, but that's only a minor inconvenience and we got used to that already.

I recommend Meteor and I'll be playing around with it a bit more for sure.

Thanks for reading!
Gergely.
