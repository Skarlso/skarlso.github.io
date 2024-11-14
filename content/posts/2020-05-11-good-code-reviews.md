+++
author = "hannibal"
categories = ["golang", "review"]
date = "2020-05-11T21:01:00+01:00"
title = "How to do a good code review"
url = "/2020/05/11/good-code-reviews"
comments = true
+++

# Intro

Hi folks.

This time, I would like to talk a little bit about code reviews.

How do you do code reviews? Don't hesitate to share it in the comments.

How do I do code reviews? Well read on if you would like to know.

# The Top Down approach

If I'm dealing with a small code change, a couple of lines here and there in the odd file
first, I'll try to understand why the review is there? What was it trying to achieve? What's
the goal of the change? Is there a ticket/issue I can read for background info? Or an RFC?

Understanding the goal of the change will let you know how to read the change. I usually also
scribble down some notes and my expectations to see if the change meets them or does something
completely different. And if it's different, maybe my expectations were wrong.

In any case, I will have a framework to start with. It's important to understand why the change
is there in the first place. I cannot stress this enough.

# Logical follow

If the change is large, the top down approach will simply not work. You will loose track of
why the change is and your logical big picture image will fade into nothingness after a hundred lines.

In Github at least, what I would do to approach this, is close all views and just have a general sense first
how big the change is, and what files changed (after I understand why the change is there and what is it trying
to change and / or solve). Once I have a feel for the structure I would look for changes which are trivial.
For example parameter changes of a function. I would expect that in that case there will be a lot of changes at places
where that function was called. I would review those and then go on.

If there is any, I would look for an entry point into the change. Is there a new handler? A new API?
A new method? Did an API change? If so, did that change propagate all the way through the API's implementation?

If it's a huge number of deletes, I would look for the deleted code in the whole codebase. Did they miss something?
Was that code referenced in another section of the code or possibly in another service? In that case, do a search
on the whole organization on all repositories if you believe that that makes sense.

If it's concurrent code... are they syncing it up at some point? Are they releasing the lock? Is the lock happening
at the right place? In Go for example, you can get a lock and then `defer w.Lock.Unlock()` it. This makes
for a convenient way of "forgetting" about the lock acquire. But this is counterproductive in some cases.

Imagine you have a function which acquires the lock in the begin. Then does a for loop which takes a couple of seconds
but doesn't actually use the map or the value you were trying to protect. In that case you are taking the
lock but aren't actually using it. There was no point in acquiring it at the beginning of the function.

# General order

There are a LOT of things one can review in a PR. Minute things and a myriad if small to big logical
problems and ramifications. It's not possible to list them all. So here are some general rules I would
follow:

## Syntax

The first thing I would do is look through the syntax and follow this mnemonic: BUD.
B(ottleneck), U(nnecessary code), D(uplicate work). Spotting these is usually easy but it can happen
that the change is subtle. Bottlenecks are often embedded loops in loops or a very sneaky recursion.
Unnecessary code is sometimes harder to spot. This is duplicate code which could be extracted. It can be subtle
because it's likely that only a small thing changes and at first glance it's not trivial how to extract
the rest of the code around that small thing. Maybe it can be a function (if your language supports functions
as first class citizens) which could close over a value and change it multiple times.

Duplicate work is when a loop is calculating something over and over but it's actually the same thing or
we already have that information and it's not likely that it would change so it can be reused. These kind of
problems are solved through caching or simply just do it once, store the result, then pass it around. Candidates
for this could be multiple calls to the same api for the same information which didn't change in between.

## General language guidelines

General language syntax and guidelines adherence comes next. In Go this is trivial, since we have a plethora
of tools available to us, devs, in the form of static analyzers like, fmt, golint, goimport etc. But in the
absence there is usually a good guide at hand how a language is supposed to look like.

## Workplace / Project guideline adherence

This could arguably come before the general adherence. Whichever suits you better. Or maybe your workplace / environment
the code is in (this could also be an open source project) is different from the general guidelines. That is okay, as
long as it's sensible. You could try changing it if you think it's too far from how a language is supposed to look like
but that usually doesn't work. Especially if the in-place guidelines are already there for years.

Generally though, it's better to follow whatever style/code/whim the current environment is doing. If changing something
always look around how that looks like in the code you are working in and then follow that style. These could be things like,
variable naming, comment semantics, logical flow of the code, structuring (like where the code should go and how it should look
like (yes, look like(sometime aesthetics matter))).


## Could it be done concurrently

As a cherry on top, I would try to determine if the work that is being done, could be done in a thread / go routine. In Go, go routines
are cheap and very easy to make. It's also easy to abuse them of course, but it doesn't hurt to think asynchronously. Especially in
a distributed environment. Which brings me to the next point.

## In a distributed environment timing is key

If this change is in an environment which has many services and is generally distributed your first though should immediately
be, how those this affect the rest of the services and what timing issues could arise. If there is a delete operation, what about
another service calling a create or a get on the same resource at the same time? What if it's a create but another service also calls
create with the same values? Is the data eventually consistent or strongly consistent? How does that affect the runtime? Is the change
in a frequently called code segment which is usually under heavy load? Did the change change the way that is handled? Did it slow it down
or speed it up? Did it trade the slowdown for strong consistency? Is strong consistency really needed in that service which would
justify the slowdown?

Like I said... a myriad of things...

I'll stop here for now.

# Conclusion

I hope this made sense. If you disagree with this approach or have a different guideline of reviewing, please don't hesitate it to share it!

As always,
Thanks for reading!
Gergely.
