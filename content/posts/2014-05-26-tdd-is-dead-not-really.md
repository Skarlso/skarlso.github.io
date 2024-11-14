+++
author = "hannibal"
categories = ["programming", "rambling", "TDD", "testing"]
date = "2014-05-26"
tags = ["tddisdead"]
title = "TDD is Dead – Not really"
url = "/2014/05/26/tdd-is-dead-not-really/"

+++

Is TDD dead?

Not really. So let's talk about this topic for a little bit.

I'm sure you already read a gazillion posts about this debate but frankly I'm writing this for myself, to rant a little bit, you know. Because somebody is wrong on the internet and I must intervene.

<!--more-->

So first of all, the hashtag #tddisdead (and I will use it shamelessly as well). This is clearly an attempt to get as many peoples attention as you can. TDD is NOT DEAD. Obviously since it has soooo many supporters how could it be dead? It's like asking, is Design Patterns dead? Or is Functional Automation dead? Or is Oreo cookies dead?

No, it's not dead. And it won't ever be dead. It will maybe change into something new, something better even, but it will never be dead. So let's skip that part.

Now, about the **debate**.

I haven't hear so much bull spoken for this long since I watched the political debate of two fractions in my home country. The right wing extremists against the left wing.. I don't know whats. And it was just that. A political debate. It had no merit and no value whatsoever. At all. Nothing.

And right in the middle **DHH** says this:

_".you're not done until you also have tests for a piece of functionality &#8212; I'm completely on board with that."._

That made the whole conversation completely irrelevant.

Every counter against TDD I heard was **bull**. Not in that debate, in general. People are either too lazy to write them, just don't want to get out of their comfort zone, don't really care about tests, or don't really care about quality or under time pressure ( I get to this later. ).

Which brings me to my next point.

**Quality**

People seem to not care about quality that much. Would they, they would understand that having a bulletproof west will save your life when you get shot in the chest with a 357 magnum. You can flush out early design flaws you can detect early bugs and do a better system design.

Sure if you are the most intelligent man on the planet maybe you can come up with a perfect system on the first draft and then implement it flawlessly so that it doesn't fall apart in two months time. But most people can't. Most people make errors on the way.

And yes, writing tests can be hard. But guess what? If writing a test is hard because that part of the system is complicated, than it will be that part of the system which will react the worst to change. And only change is constant. Which brings me to the next item.

**Time constraints**

So your manager is sitting right next to you and saying come on we are paying you to write code and not tests so do it! And you have to have a feature done today but if you write a suite of tests you'll only finish tomorrow. Sure, your estimate at that point will become a very quick one because you make a sacrifice of trust.

And then the next story comes along and you say. _"Sure I can do that as well. No problem I know how my system works, right? Hmm. why the hell did that break all of a sudden? I didn't change anything in that module. Ahh damn it I said I'll be done today, so let's just fix this quickly and then move on to the next card."_

And the next story comes along. _"Sure I can do that. wait a minute. Didn't that part brake already twice? Damn, better refactor. Ohh shit, why is that now breaking???? Damn it I said I'll be done tomorrow, better patch it, and then move on. Hmm let's write a test here to make sure this does not break. Ohhh damn I need PowerMock for that stuff since it's in another module. Why the hell is that there? Should it be here in the first place since it's somehow used by that other class there? Interesting. Let's refactor and put it in here so I can mock it. Ahhhh f\*ck now all the rest of the system is not working. Damn, I'll just use PowerMock. Shit. Checkstyle error. PowerMock is not allowed?? What?? Who the f\*ck says that?"_

You get my drift. And suddenly you end up with estimates of **WEEKS**!!!! instead of days / hours for a simple story.

**Finishing it up**

This a rant only. It's my personal opinion, experience and observation of a 10 year time period in Software Testing. Starting with at least a Weak Skeleton and a few upfront tests will help you in the long run. Writing at least ONE - TWO acceptance tests WILL help you understand business logic better. Writing ONE or TWO unit tests will help you understand your logic better. I'm not saying write a whole damn suite of tests I can understand you don't want to do that, but for quality's sake write at least a couple.

You will love it, I promise you that.

Thanks for reading.

Gergely.