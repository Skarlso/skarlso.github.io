+++
author = "hannibal"
categories = ["books"]
date = "2021-11-27T01:01:00+01:00"
title = "Summary of Programmer's Brain"
url = "/2021/11/27/summary-of-programmers-brain"
comments = true
+++

# Prologue

Hello all.

This is a summary of the book [Programmer's Brain](https://www.manning.com/books/the-programmers-brain). Let me begin by saying the book is fantastic and you should definitely read it. The research put into it is phenomenal, and the many linked notes, data and reference make for an amazing and compelling read!

Not to mention that it is fantastic that someone put actual effort and data into finding out how programmers operate, think and behave. And most of all, how our code behaves and reading, and dealing with code is not just about understanding algorithms and finding the best possible and most optimized code ever.

What does this summary contain? Information that I found useful about the book. It doesn't contain everything, just the distilled amalgamation of information that I hand picked and will try to convey using my own words.

Let's dive in.

## Part One - Reading code better

### Understanding

We all know that we read code more than we write it. It's a basic premise, right? Yet, still we don't write code for reading at all! We have to make a conscious effort of it to write it in a readable, compact way. We optimize for speed, optimize for structure, but we rarely, if at all, optimize for readability. For understandability. To please the reader, instead of the writer. To look ahead an think, by whom and how will this code be read again?

This part offers a couple of solutions to alleviate this pain.

#### Confusion

It identifies three things. Lack of Knowledge, Lack of Information and Lack of Processing Power.

- lack of information - basically, you don't know enough about the project / code you are looking at
- lack of knowledge - you lack knowledge about the thing you are doing, i.e.: some obscure language syntax you are unfamiliar with
- lack of processing power - the code chunk you are looking at is too large, or you should break it down

The book makes a point on how cognitive load works and why you should take care of it. I'm not going to detail that part ( read the book! ). But it's basically all about how complex something looks until you have enough background in your long term memory to be able to decipher it instantly and not spending too much time on it.

*Lack of processing power* is alleviated by taking notes, jotting down information and functions and variables about the code you are writing. By using a second brain (i.e. paper) you lighten the load on your current working memory. Further, it might be helpful to refactor into something that you do understand and are already familiar with. For example, exploding a Python list comprehension into a for loops until you understand the whole thing. Then put it back and read it again.

*Lack of knowledge* is easy to mend. You read up on said topic / become better in it.

*Lack of information* again, just read up on the stuff you are missing.

#### Reading

The book then continues with helping you speed read code better. There are fascinating researches quoted about various things they tried with experts and junior programmers and chess masters vs beginners.

As much as Go people dislike talking about design patterns in Go, they exist for a reason. The reason is that there are common problems which have common, tried solutions. It's fare that these patterns apply differently in Go because of the structure available to us, but they still exist. And this is on which speed reading hinges. Patterns. The more patterns you recognize the faster you'll be able to read code.

`for` loops, `if err != nil` checks. It's ingrained now into your memory and you recognize and can read it at a glance.

This means, in order to help the reading of your code, you should focus on things called Beacons. Beacons are small, understandable (at a glance) parts of your code. Comments, patterns like `if err != nil `, chains, proper function names; all of these are beacons which help in understanding and reading your code better.

For faster code reading speed, you have to know patterns and idioms at a glance. Practice until you know them by heart. For example, you don't have to look twice to identify a bridge above water, right? You know it's a bridge because it looks like a bridge. It could be above a street too it still would be a bridge.

To be able to learn faster, the book, and indeed many other research papers now-a-days, suggest spaced repetition, recall, and / or flashcards.

One of the best suggestions I've gotten out of this book about reading and learning code, is if you have to Google something for the second time because you don't remember how that works, you are wasting time and are adding to your cognitive load. At that point, you should make an effort to remember it with the above suggested tips. Why, even for the first time. If you Google something or look it up on StackOverflow, at that moment, you should create a flash card for it and condense that information WITH YOUR OWN WORDS! Then use spaced repetition / recall to remember it.

In the future you will benefit from having that thing in the back of your head.

## Part Two - Thinking about code

### Reading strategies

Then we get to reading and understanding code. Research backed data proves that mathematical knowledge will not make you better at understanding code. Linguistic proves backs programming skills more than mathematical understanding. Logical proves also helps a lot more.

Effective reading in general, can be broken down into the following sections:

- Activating -- try remembering things which relate or are similar to the thing you are reading
- Monitoring -- check your understanding as you go
- Determining importance -- discard things that are of no interest to you
- Inferring -- understand things that the text only implies
- Visualizing -- draw things -- only helps if you are visually inclined. I think you should do whatever helps you to understand
- Questioning -- without recall there is no remembering
- Summarizing -- write a summary IN YOUR OWN WORDS!!! ( I can't stress this enough )

When you are reading code, you first will understand the code at hand, and then, a bigger picture understanding comes later. I'm not sure about which comes first. I think everyone is different. Someone might understand the bigger picture a lot easier than actual code. In any case, mapping either to the other is a difficult process. Inferring code from patterns and inferring the bigger picture from code. It's a good exercise.

The book also separates variables based on behavior. This might help you understand the code better later, when you read it. It uses the following 11 categories:

- Follower
- Stepper
- Walker
- Temporary
- Counter/Gatherer
- Container
- Fixed value
- Flag
- Organizer
- Most-recent Holder
- Most-wanted Holder

These are self-explanatory, but marking the variables might give you a hint on their purpose and maybe refactor them or name them appropriately.

Now, we come to the interesting sub-section of problem solving.

### Problem solving

#### Mental Models

The way you create a mental model about a problem will decide how you'll go about solving them. Which means, if your solution "feels dirty", or is buggy and doesn't feel right, you just might need to change your mental model about the problem. Or the data representing it. Your mental model will decide how you tackle this problem. So you better make sure your mental model is correct. A couple of mental models can be, diagrams, a graph, a state machine... everything that can represent the logic of the code with a model.

Thinking with models about the logic of the code is an important skill. Abstracting the problem can give you a huge benefit in solving it.

There is an awesome quote from Rob Pike:

> A year or two after I’d joined the Labs, I was pair programming with Ken Thompson on an on-the-fly compiler for a little interactive graphics language designed by Gerard Holzmann. I was the faster typist, so I was at the keyboard and Ken was standing behind me as we programmed. We were working fast, and things broke, often visibly — it was a graphics language, after all. When something went wrong, I’d reflexively start to dig in to the problem, examining stack traces, sticking in print statements, invoking a debugger, and so on. But Ken would just stand and think, ignoring me and the code we’d just written. After a while I noticed a pattern: **Ken would often understand the problem before I would, and would suddenly announce, “I know what’s wrong.” He was usually correct. I realized that Ken was building a mental model of the code and when something broke it was an error in the model. By thinking about *how* that problem could happen, he’d intuit where the model was wrong or where our code must not be satisfying the model.**
>
> **Ken taught me that thinking before debugging is extremely important. If you dive into the bug, you tend to fix the local issue in the code, but if you think about the bug first, how the bug came to be, you often find and correct a higher-level problem in the code that will improve the design and prevent further bugs.**
>
> I recognize this is largely a matter of style. Some people insist on line-by-line tool-driven debugging for everything. But I now believe that thinking — without looking at the code — is the best debugging tool of all, because it leads to better software.

I love this. It shows that having a clear mental model of the code will help you in understanding it better. Work through the code, construct a model by following a call path. Understand and draw the entities which interact with each other through that path and construct a structure out of it. Draw that structure or put it into a state machine, write down how pieces and variables interact and you'll have a clear understanding of what's going on.

This is called following a focus point. A focus point is an entry to a system. Sometimes there are multiple focus points. They all represent a different path through the system but might use the same entities.

The more general mental models you store in your Long Term Memory, the easier it will be to identify and solve problems. Use flashcards to remember as many mental models as you can. Things like, tree traversal, sorting, data structures, thinking about architecture, layered architecture etc... If you can recall them from heart you'll be able to reason and understand code a lot better.

#### Notional Machines

The best way to describe a notional machine is an abstraction of the executing code. It's a kind of mental model, but it's always correct. A normal mental model can be incorrect. A notional machine always depicts what the machine is doing. I didn't really find them helpful, but they might be helpful to others. Notional machines help us understand how the machine works, how code runs. There are multiple machines which can work together. To learn more about them, either read the book, or read [this](https://www.researchgate.net/publication/259998496_Notional_Machines_and_Introductory_Programming_Education) research paper on them.

#### Misconceptions

This one is interesting and certainly worth an explanation. Basically, you have to constantly check you biases about code, concepts and your mental models. Otherwise, you'll fall into the routine of using the wrong model all the time. This isn't always easy, because it might not be apparent at first. But never assume that what you know is always correct. Of course there are axioms like `1==1` which will always be true, but that doesn't mean that a concept you might know is an axiom.

Always read the documentation and the code of the current project you are working on with the care of checking your mental models. They might be outdated or incorrect.

## Part Three - Writing better code

### How to get better at naming things

This is a whole chapter in it's own about naming things. That is an important chapter to be sure because naming is the hardest. But I found that the advice there is basically, name things in the context they are in and regarding the behavior they represent. Naming things has always been important. A good name will sometimes solve the whole problem in its own!

There was one time where I was thinking about how to implement a particular feature. I've spent a day on trying to come up with a nice solution. When I stepped back and thought about the problem without looking at the code. Building up a mental model, thinking about what I need and if there is a `thing` that can do that for me. When suddenly, the best name popped into my head based on the need, the circumstance and the behavior of the thing. Suddenly, the whole implementation laid itself out before me because I knew what I needed. I had a name for it.

Equally, a bad name will deprive the understanding you need and, in fact, will make it much harder to implement anything else in that context.

Amongst other things, the book suggests the three-step model. [How Developer's Choose Names](https://www.cs.huji.ac.il/~feit/papers/Names20TSE.pdf).

### Avoid bad code and cognitive load

First, let's look at the cognitive load part. What does cognitive load mean? It means that you have to spend a lot of brain power / energy on understanding the actual code. Why is that? Is the code way too complex for what it's doing? Is it written badly? Does it smell? Do you not understand the domain? Do you not understand the syntax? There are many things you can do to alleviate some of the pain. The main three are:

#### Make sure you completely understand the domain

This one is easy to dismiss but is actually hugely important. You have to be absolutely crystal clear that you understand the domain that the code is dealing with. Otherwise, the code will either not make sense, or will be difficult to interpret in it's fullest. By understanding the domain, you'll be able to figure out the code, even more so, you'll be able to reason about the code. And maybe you'll actually find that the code is in fact, wrong. It doesn't actually implement the domain well.

#### Simplify the code

Sometimes the code is just written in an insane complex way for no apparent reason. Or there is a reason, you just don't see it yet. And that's okay! Start by refactoring the code in a way you understand or making it simpler. I, actually have a story about this.

There was a [ticket](https://github.com/weaveworks/eksctl/issues/1275) in [eksctl](https://github.com/weaveworks/eksctl) I picked up. Refactor the logging to expand nested task logging into a nice structure. Basically come from this:

```
[ℹ]  3 sequential tasks: { delete nodegroup "ng-5b8f2f94", 2 sequential sub-tasks: { 2 sequential sub-tasks: { delete IAM role for serviceaccount "default/s3-reader", delete serviceaccount "default/s3-reader" }, delete IAM OIDC provider }, delete cluster control plane "irp-test-5-manual" [async] }
```

to something like this:

```
[ℹ]  3 sequential tasks: {
  delete nodegroup "ng-5b8f2f94", 2 sequential sub-tasks: {
    2 sequential sub-tasks: {
      delete IAM role for serviceaccount "default/s3-reader", delete serviceaccount "default/s3-reader"
    }
    delete IAM OIDC provider
  }
  delete cluster control plane "irp-test-5-manual" [async]
}
```

Looks easy, right? This was the result: [pr](https://github.com/weaveworks/eksctl/pull/4290). It took me two days! Turns out what parsed the log messages was so convoluted and recursive that it was difficult to unravel. Ultimately, what I did was, I begun re-writing bits of it and simplifying the whole thing, running the code over and over to see what it produced. It was a loose, semi-recursive structure, where you had no idea how deep you are at any given moment; thus you couldn't just indent things based on a counter or a tracker. It was always 1 or 0 deep.

Even so, I think this will sort of break apart if we ever go 4-5 nested tasks deep... but that's a story for another day.

The lesson here is, that simplifying and re-writing the code made me understand what's going on there a lot better and I was finally able to rewrite it.

Next, the book talks about how code smells increase the cognitive load many fold. Eliminating code smells will increase the understandability of the code. Code smells have been introduced by Kent Beck and become more known when Martin Fowler wrote about them. A brief description can be found [here](https://martinfowler.com/bliki/CodeSmell.html). The `Refactoring` book identifies these code smells:

- Long Methods
- Long Parameter List
- Switch statements
- Alternative Classes with different Interfaces
- Primitive obsession - Avoid the overuse of primitive types in a class
- Incomplete Library class - Methods should not be added to random classes instead of to a library class.
- Large Class
- Lazy Class - doing too little
- Data Class - just dealing with data
- Temp fields
- Data Clumps
- Divergent Change | Shotgun surgery - change which results in unwanted change in different parts of the code
- Feature Envy
- Inappropriate intimacy - classes should not be connected to other classes extensively
- Duplicate code or code clones
- Superflouse comments
- Message Chains - aka. train wrecks
- Middle man - its only purpose is to delegate to other method calls
- Refused bequest (inherit unused behavior)
- Speculative generality - adding code "just in case"

Some of these don't really apply in Go, especially the pieces about classes and inheritance. But some of them do apply and should be looked out for. They can really complicate the code unnecessarily. Further anti-patterns are Structural and Linguistic.

**Structural**: Code is okay, but structured in a way that is difficult to understand and process -> increases cognitive load.
**Linguistic**: Bad / misleading names lead to confusion and bugs.

### Solving complex problems

First, identify what the problem is. Re-write / re-formulate it in your own words. This will make sure that you understand the problem at hand properly. Then draw the problem out as much as possible and identify the individual constraints which exist. List them and examine them. If the problem is too complex, start removing or adding constraints until it is easier to think about the solution.

The book talks about State Space. A state space is all steps we could consider when solving the problem.

Now, the book says something I would like to quote word for word, because I believe this is profound and should be taken to heart.

> In 1945, Pólya wrote a short and famous book called How to Solve It. His book proposes a “system of thinking” to solve any problem involving three steps:
    > 1. Understanding the problem
    > 2. Devising a plan
    > 3. Carrying out the plan
>
> However, despite the popularity of generic approaches, research has consistently shown that problem solving is neither a generic skill nor a cognitive process.

Let me repeat this. **However, despite the popularity of generic approaches, research has consistently shown that problem solving is neither a generic skill nor a cognitive process.**

There is no such thing as learning a base generic method of problem solving and then solve all possible problems. You have to exercise specific problems in order to be better at specific problems. Guess what this means on a broader aspect. Learning to play Chess will not make you smarter. It will make you good at chess. Playing games on Lumosity will not make you smarter. It will make you better at playing games on Lumosity.

What will definitely help you though is the following. "Automate" mundane algorithms. What this means in practice is that you exercise problems which include things like, tree traversal, graph building, stack based operations, breath first search, depth first search, etc. Practice them until you know these things by heart. The same way you learn to do math. You solve quadratic functions until you know by hearth what the solution is.

But you don't stop there!!! You also need to be able to evolve these. Apply them to similar problems and understand how a problem is similar, or break it down until you can make it similar. I'd like to think about like when Miyagi thought Daniel son **wax on wax off** technique and then suddenly, Daniel knew how to do karate. That's a bit of stretch but you get the gist of the idea.

The main is that common things like, what is an optimistic or a pessimistic lock, have to be in your long term memory. Continuously Googling and looking at StackOverflow will not make you better at problem solving. That does not mean you shouldn't!!!!!!!! BUT! You should eventually learn from it! And now, let me repeat an important sentence. **IT IS OKAY TO LOOK AT SOLUTIONS FROM OTHER PEOPLE!!** Like, in art. You first learn by coping things. Things from real world, other artist's work. You study art from other people and drawing from masters. You understand how they did it, and replicate that until you know it by heart. On then do you begin to extend and find your own style.

And so, it's perfectly fine to look at other people's solutions to hard problems **AS LONG AS YOU LEARN FROM THEM**! Look at the solution. Study it. Understand it. Then write your own! That's how you'll learn.

The book goes into a lot of detail about what kind of memory influences these problems. But this, the above thing, was my biggest takeaway from all of that. Basically, automating things in your brain will free up cognitive load when thinking about code and solving problems.

## Part Four - Collaborating on Code

### Writing code

The book first starts off by listing all the different activities when writing code and sums them up into the following sub-sections:

**Searching**

Mostly reading and learning the code. Hard on short term memory. Helps to write down where you are, what you are looking for and what did you already found to off-load cognitive load. Leave breadcrumbs, change the code here and there, leave a TODO / anchor.

**Comprehension**

58% of our time is spent doing this activity. It's about understanding the code and trying to find out what it does and how. Changing / simplifying the code helps. Draw models and leave notes on structure which can help you continue your task later.

**Transcription**

"Just coding" where you know what you are going to write where and why.

**Incrementation**

Adding new feature. Is hard on everything...

**Exploration**

The final activity. Sketching with code. Vague idea but by writing code, your idea solidifies. Jot down your design, will help order your thoughts and free up mental space for other things.

Then the book continue to talk about interrupts and how they are not really helpful. We already know this...

### Improving the design of large systems

This is an interesting chapter I completely misunderstood but I'm glad I got perspective on. It makes you think about your system's design in terms of cognitive load and how hard it would be for other to find their way in it. Basically design your system with humans in mind. Is it easy to follow? Do package names make sense? Does that code have to live there? Your architecture might be awesome, and intricate, but if others can't read it, then it they won't collaborate on it. And if you are working in a team this is especially important. The book calls this CDCB ( cognitive dimension of codebases ).

CDCB identifies the following dimensions:

**Error Proneness**
This one is mostly about type safety and how easy it is in your code to make errors.

**Consistency**
How consistent is your code with itself. Naming, structure, where and why things are where they are. Paradigms, idioms being used...

**Diffuseness**
How much space a programming construct takes. How long certain required syntaxes are. Things like for loops, and language structures. How compact or extent the code is.

**Hidden Dependencies**
How visible are the code's dependencies. Are there hidden things the user needs but is not apparent and not documented?

**Provisionality**
Exploring ideas can help when writing code. You start writing and that will give you ideas about how to proceed further. Your vague ideas become more concrete. Some language can be used better to do this because they don't get in your way. Like Python, JavaScript. Go is strictly typed and will scream at you if you make a mistake. That can hinder exploration.

**Viscosity**
How resistant is your code to changes.

**Progressive Evaluation**
Similar to Provisionality, this dimensions defines how hard it is to execute partial code.

**Role expressiveness**
How easy it is to see what does what. Like, in Ruby it's difficult to see if something is a function call or a variable. `thing.other`. Other might be anything. Meanwhile in other languages it's easy to see that `thing()` is a function.

**Closeness of Mapping**
How close your program is to the domain / problem that is solved.

**Hard Mental Operations**
The code requires hard mental operations, such as, deciphering obscure syntax, or understanding hard idioms like monads and difficult to follow type systems. If your project requires hard mental operations, contributors are less likely to join it.

**Secondary Notation**
This one is weird. Second meaning to code, such as comments and named parameters.

**Abstraction**
Can users of your system create their own abstractions which are as powerful as built in abstractions. For example functions in a programming language. In Go I would say interfaces provide this abstraction since a user of a Library can create their own Interface over some functionality of a library and use that and pass in the libraries function.

**Visibility**
How easy is it to see parts of the system. Is your folder structure following your architecture layers?

Playing around with these and improving one or the other might improve the overall design of your project. But there are trade-offs for everything.

### How to onboard new developers

This chapter is all about the differences between senior and newcomers and ties nicely back into all of the research being done about cognitive load, short term vs long term memory, sensory overload and information overload. It's an interesting read for sure if you do a lot of interviews I definitely encourage reading the whole thing in the book. I'm going to summarize what I've got out of it...

**Don't overload newcomers.**
They need to learn a LOT about your environment, group dynamic, team procedures, issue handling, break routines... and I didn't even begin talking about code yet...

**Once you master something you forget how difficult it was.**
This is a great point we tend to forget. You have a better view of the big picture. You know parts of the code better. Beginners tend to focus on detail more and don't see the big picture yet. You tend to think that that stuff is "easy". Since you already forgot how difficult it is/was.

**Noticing patterns.**
Even if you don't immediately have an answer to a problem you'll be faster in finding it. You can chunk code better, you can read code faster and spot certain patterns more efficiently since you already encountered them a hundred times over.

**Semantic Wave**
This one is a really interesting concept. It depicts the ideal learning curve of a newcomer. Coin by Karl Maton. Read about it [here](https://www.sciencedirect.com/science/article/abs/pii/S0898589812000678).

Has three steps...

1: *understanding* - understand the generic concept

2: *unpacking* - this is the low of the curve where the beginner is ready to learn the details about the thing

3: *repacking* - get back to the generic concept on their own - meaning understand why the generic concept is useful and results in the concrete detail. This is when the information is actually internalized and ideally, completely remembered.

Following this there are three anti-patterns when not regarding this curve.

1: *high flatline* - only using abstract terms

2: *low flatline* - too much detail, not enough high level view on the **why**

3: *funny enough this one doesn't have a fancy name* - starts with the abstract, goes to the detail but doesn't allow time for the internalization, the *repacking*. Let's call this one *the rush*.

The book has some more awesome suggestions and activities to follow with the newcomers, but I can spoil all the fun now can I.

## Conclusions

This has been it. Thank you for reading. I hope this was as fun to read as it was to write. It was a great journey reading this book and the citations and research that went into it.

Thanks for reading,
Gergely.

