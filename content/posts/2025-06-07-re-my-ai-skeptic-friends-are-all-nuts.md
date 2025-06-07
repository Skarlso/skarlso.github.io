+++
author = "hannibal"
categories = ["ai", "llm", "delve"]
date = "2025-06-07T01:01:00+01:00"
title = "Re: My AI Skeptic Friends Are All Nuts"
url = "/2025/06/07/re-my-ai-skeptic-friends-are-all-nuts"
comments = true
+++

# Re: My AI Skeptic Friends Are All Nuts

There was a post recently, that was [dissing AI Skeptics](https://fly.io/blog/youre-all-nuts/). And while the post is funny at times
I feel like it's absolutely and completely missing the point of the skepticism. Or at least I
feel that it is glancing over some massive pain points of said skepticism.

Let's go over some of the points and the things that stuck out to me that are problematic.

Now, mind you, I don't necessarily disagree here. Like, an AI review helper on an
open source project where maintainers are swamped with tickets, would be a godsend ( assuming 
its review is actually valuable and can be fine-tuned on the ever changing codebase ).

But us humans, shouldn't outsource our critical thinking completely. It will deteriorate so fast
you blink and suddenly, you have no idea how to do a simple lookup without firing up your LLM
agent.

>I can feel my blood pressure rising thinking of all the bookkeeping and Googling and dependency drama of a new project. An LLM can be instructed to just figure all that shit out. Often, it will drop you precisely at that golden moment where shit almost works, and development means tweaking code and immediately seeing things work better. That dopamine hit is why I code.

My dopamine comes not from tweaking someone(something) else's code to make something barely work. My dopamine
comes from the fact _that I solved some intricate none-trivial problem_. Sure, write me unit tests
that's awesome. Or write that fifth table join + query code, go for it. But nevertheless the part where
_can be instructed to just figure all that shit out_ is extremely dangerous. I cannot stress this enough.
You might not like it, because you are an advanced user, but a young person just coming into the
craft, absolutely needs to _figure all that shit out_ otherwise all their experience will be something
akin to _oh I'll just ask my agent to do that for me_. That's not going to cut it.

And try to see if in an interview you can say, oh I have no idea, I'll just ask my llm. The problem
here is not you using a tool to further your goals. The problem is that most people begun outsourcing
their critical thinking and problem solving abilities. And it shows. Even for me it shows. I tried
to write some test code recently and I absolutely forgot how to write table tests because I generate
all that. And it's frightening. I haven't been using LLM for long. And you can't compare this level of generating
with snippets or, or StackOverflow or copy and pasting. This is on a different and more sophisticated level.

Now imagine what's going on faculties and schools and researches where your ability to think is now
something like _I'll just ask the LLM_. Of course there are always out-liners. People who use it like
an assistance. But those are the minority. The majority needs guardrails otherwise there will be a harsh
tipping point where suddenly people will no longer remember how to structure code, or design a simple
application.

>Also: let’s stop kidding ourselves about how good our human first cuts really are.

This and the rest of the article is ignoring a couple of points that are truly bad.

- If you stop investing into the _human first cuts_ where do you think the industry will end up? Dead in a ditch.

Sure, this was always an issues. No-one wants to invest into starting people. But at least until now, you _had_ to. But
if you don't that's it. If no-one is training and hiring beginners there will be no seniors. Give it a couple generations and
you will have people switching away from dev since a ~$20 / month LLM apparently can do that same thing ( which is
absolute bullshit ). And then suddenly, there are no more devs around. And maybe that's the future. Maybe
robots will write all the code and there will be only a handful of people maintaining them until they
are also gone. But that's sci fi. Hopefully.

- LLM is training on LLM and agents can't help with that. There is, already ( which is insane since it has been around
for only a couple of years ) a significant increase in AI slop in research papers and online and blogs, etc. Just look up
how `delve` has increased in usage. And, of course, since LLMs are trained on online data, it is re-trained with its
own ai slop further damaging it. Soon bad code will get even worse basically.

- Programming is art too. Indeed, architecture is sometimes very much an art form. Technical solutions to problems sometimes
require lateral thinking. And solutions to hard problems might even come from an activity completely unrelated to the problem.
There are many examples where a solution to a problem came from thinking specifically _not_ like an engineer. Pixar's shaders,
the first UIs, the invention of Hypertext, Smalltalk!

- And yes, it is stealing.
The amount of plagiarism and the amount of ART that LLMs have stolen is insane.


> hallucinations

To this the author suggests reading the code. Sure, you are an experienced developer who read and wrote many lines of code.
However, you are missing the most important problem here. Somehow who is still learning will be unable to tell if it is hallucinating
or not. Because it reasons like my drunk friend ( with absolute conviction ) people tend to accept it. They either yet lack the
ability to think critically and try to look for different solutions, or they are just tired and burned out by the rest of the
things they need to care about and they will say the dreaded line _it looks good enough_.

## Conclusion

So am I saying don't use LLMs? No, I'm not. I'm saying it needs serious guardrails, documents,
oversight, usage data, research and it should definitely be banned from school. Because it's doing serious
damage to young people's ability to think, figure out, read, understand and research.

Further, the person says they are a _serious developer_. This might actually be a drawback in this situation.
They assume that people will think critically, will vet the code and will understand when its flawed. Well they
don't. Most of the time they just don't. And reading ai slop for the 100th time, because IT WILL NOT LEARN is
just tiring. While if I read the code from a junior dev, and tell them for the 5th time that something is wrong
they might actually make an effort and understand and next time, maybe not make the same mistake.

An LLM will 100% make the same mistake. Over and over and over again. Even if you have an Agent and do some
serious training with marginally small loss. It still still stumble on conceptually the same thing. I'm sure it will get
better with time, but it's not there yet. And saying and accepting it like it is, is very dangerous.

And as a last. I fear very much what's going on in schools. My son is in school. And the teacher literally said
they need to use and LLM otherwise they won't score high enough because the other pupils _are_ using LLM and the
teacher is using an LLM to grade / check it. This is ridicules. And if you think that's okay or you don't see
a serious problem with this, then that's an even greater problem.

As always,
Thanks for reading!