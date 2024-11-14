+++
author = "hannibal"
categories = ["programming", "rambling", "testing"]
date = "2013-11-02"
title = "Why you should care about the quality of your test code"
url = "/2013/11/02/why-you-should-care-about-the-quality-of-your-test-code/"

+++

Hello folks.

Today I would like to talk to you about something interesting I was talking about with a developer friend.

We talked about the quality of test code.

He said. And I will quote this."Why should we care? It's not production code. We aren't giving it to the customer."

There are a few reasons why you are going to get a slap in the face for a sentence like this. And let's clarify here that we are talking about unit and functional tests as well. It shouldn't matter what tests you are thinking about.

**Reason for a Slap #1**

If a new comer comes to the company ( and don't tell me that's not happening so frequently ) then there are a few very good ways how he can learn to work with the new system. The first and almost best way to do so is. to look at the tests. Because the tests are representing your system. And how will it look like if the tests are in a bad shape? What will his or her thoughts be?

a. Wow what nice people what nice code. This looks fantastic. I'm sure they are a bunch of people who care very much about code and practices and the quality of the product.

b. Wow this is amazing. I'm sure I will learn a lot about good coding practices here and I will have fun with a bunch of very clever people.

c. Wow what the hell is this piece of cr*p? How the hell did I end up here? What are these people? A bunch of neanderthals? What does this even do? Where did it come from and why?

I think we can agree on what his thoughts will be. And on top of that he will have a very hard time learning the code and what it does.

**Reason for a Slap #2**

Another reason is because you think that you write it down once and then you can forget about it. Well guess again. That's not how things work in the software development word. You WILL have to go back to it eventually and then you will curse all hell for begin such an idiot about it not to factor out that one method that would have made your life, and everybody elses, a bit easier.

Even after a week or two you won't remember how and why you wrote what you wrote and then you will be in a whole new world of hurt. You will have a very hard time finiding out the things you did and then trying to backtrack your steps to a place where you have some recollections.

**Reason for a Slap #3**

It's like grammar. You think it doesn't matter that you misspelled a word or two in an error message. Or that you have a bad name for a method or a really really critical grammatical error in a catch sentence? You think it doesn't matter since it's not affecting the logic of your code? Well think again. You are right in that it doesn't affect the logic of your code ( as long as you constantly make the same grammatical error in a sentence ) but it will affect how YOU personally look like.

It will affect your profession. The way people think and talk about you. They won't think that you are a professional even though your logic is solid. They will think that you are sloppy and careless. And the same goes for the quality of your tests.

**Reason for a Slap #4**

Quality can determine the solidness of the logic in the test. If your quality is bad you might actually test the bad thing. Your test might actually not do what you think since you can't even figure it out. Your test might be doing something entirely different and you wouldn't even notice.

And a fautly test leaves you with a false positive and a potential very serious bug on your hand which you thought you had covered.

**Final Slap ( I mean thought )**

So. the quality of your tests, even if you won't give them to your customer, matter for you. They matter for your company, your image and your fellow developers, testers. They will determine their view of you who wrote them and of your abilities in ways you didn't even think of.

Please care. Save a test or two. Donate to the Test Trust Fund(tm), TTF today. Call 555-12234-Slap and be the one who cares.

As always, thanks for reading,

Have a nice day!

Gergely