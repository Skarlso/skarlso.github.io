+++
author = "hannibal"
categories = ["programming", "testing"]
date = "2014-05-23"
tags = ["javascript"]
title = "Five reasons why a front-end tester should learn Javascript"
url = "/2014/05/23/five-reasons-why-a-front-end-tester-should-learn-javascript/"

+++

Hello everybody.

Today I would like to write about a very interesting topic, I hope. So let's get started.

As the title already suggests, I'm writing about why a front-end tester should learn at least a little bit about JavaScripting and the DOM.

Ohhh and contrary to the belief CSP ( Content Security Policy ) will **not** be the death of such scripts. There are white-lists and workarounds and exclusions which can be implemented in order to allow local JavaScripting to continue. So don't fret. Read on.

<!--more-->

**Reason Number 1: Injection**

Every front-ender tester has a waste amount of tools at their disposal. Various things, like Firebug and web developer Toolbar and. Bookmarklets and <a href="https://addons.mozilla.org/en-US/firefox/addon/greasemonkey/" target="_blank">Greasemonkey</a> and <a href="https://chrome.google.com/webstore/detail/tampermonkey/dhdgffkkebhmkfjojejmpbldmpobfkfo?hl=en" target="_blank">Tampermonkey</a>. These are the real powerful tools though. Your main objective at this point would be to inject something into your current site.

Suppose a scenario where you are testing something and you have an API at your disposal for a quick edit on a customer, like closing his account or giving him money, or doing something with his appliances. Suppose you don't want to always switch away to another tab or call a service with certain parameters or go to a rest client and initiate that from there.

Suppose you could have a small injected DIV right on the page you are which gathers information for you, like the customer's username, and wallet and whatnot, and with a simple press of a button, you just closed their account. Neat, isn't it? Simple, fast, very comfortable with just one press of a button.

Suppose you have a DIV at the corner of a page, or even a drag and drop enabled one which you can drag around, with an arsenal of these buttons RIGHT THERE on your page. You don't have to switch context at all.

These days it's especially easy with tools like jQuery at your disposal. You just inject jQuery first, if the site is not already using it, and you are good to go and do whatever you like to do..

**Reason Number 2: Data gathering**

While we are testing these application we always create some kind of a report. That report contains many things about the customer, or it's appliances or the things he does, does not. All these could be constantly gathered by your script as it runs in the background. It gathers statistics and information which otherwise you would have to gather from some transaction history, or some kind of an action history. But no. Not the JavaScript Wizard.

You, would just press a button and the script would generate a report for you. It would even print it out. Create a persistent layer in which you are gathering information continuously. Create a small mySql database on your local machine and have the JavaScript enter data into that. Tadaaam.. Usage statistics at the touch of a button. All there, only waiting to be extracted.

**Reason Number 3: Tempering**

In understanding the ways of the DOM and the JavaScript you can create some very interesting test cases not to mention XSS attacks which is essentially JavaScript Injection. That's always fun and produces many very good bugs.

Cookie manipulation. You want to simulate something? Like a time-out or a session loss or anything like that with a push of a button? Easy.

**Reason Number 4: Shortcuts**

You have a massive field like registration that you need to fill out? The shortest way is to have an API which you can call via a curl script. But if that's not available and you would like to exercise the front-end any ways, then you will end up wasting hours and hours on always filling out all of those pesky fields.

And suddenly I'm hearing: "But I'm using Selenium plugin for that." - you might say. Sure, use that. But I'm using Chrome. "But there is iMacros for that." - you might say again. Sure, I know. But! Let's see which takes longer.

Open selenium, load the script, run it, see it fail, run it again, ahh success, good. Same with iMacros. As opposed to, having a Bookmarklet right in front of you, on your bookmark, and with a click of a button, or with entering something into the browsers search bar, you suddenly fill out the form and press submit.

You see the difference is that JavaScript runs faster and more accurate by it self in such short things then with a wrapper around it. And it's faster accessible as well.

**Reason Number 5: Security**

There are all sorts of things that a security tester can do with a small script which gathers session information and availability.

**Reason Number 6: Accessibility**

This is of course the easiest one. There are ample of scripts and browser plugins to test accessibility which is an all time favourite for everybody in the front-end land. Make your life a little bit easier. How bout just running a bookmark like [HTML_CodesSniffer][1] an see a very gorgeous result like this:

![][2]

Ain't it beautiful? What stops YOU from writing your own?

So get out there and learn JavaScript. It's easy. I'm not telling you to become a front-end developer, just know thy tools and you shellet receive the blessings of the IT Gods.

As always,

Thanks for reading.

Gergely.

 [1]: http://squizlabs.github.io/HTML_CodeSniffer/
 [2]: http://i1-scripts.softpedia-static.com/screenshots/HTML-CodeSniffer_1.png