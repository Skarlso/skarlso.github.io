+++
author = "hannibal"
categories = ["ai", "claude"]
date = "2026-02-17T01:01:00+01:00"
title = "How to use ai efficiently without going brain dead"
url = "/2026/02/17/how-to-use-ai-efficiently-without-going-braindead"
comments = true
+++

I know I [spoke out](https://skarlso.github.io/2025/06/07/re-my-ai-skeptic-friends-are-all-nuts/) against AI in the past, and I still think those thoughts are warranted. That's why I would like to write down how we actually should use AI to do the things we want to do without losing our critical thinking in the process.

Here's my flow...

## How to use LLMs responsibly

There always needs to be feedback. Don't use it to think, use it to ask questions, verify, check, expanding your understanding, your knowledge.

And what I like about it is that you can ask it stupid questions all the time every time. It won't judge you.

If you just keep using it without thinking about what you are trying to do for a moment, you will get into the situation where, when your LLM is down, you are useless.

Try to avoid that.

What you want to do is use it as a rubber duck. Try bouncing ideas off of it. Try telling it things like: _"make me think"_, _"ask me questions if I'm sure that I understand what it's trying to do"_.

Let's walk through a few scenarios...

### How to use it to review code responsibly

Take, for example, the following pull request on external-secrets operator project:

[Nebius MysteryBox integration](https://github.com/external-secrets/external-secrets/pull/5868)

This is a VERY large pull request with thousands of lines of change. Without an llm assisted code-review, it would have taken me MONTHS to review this thing.

With the LLM assistant it took me a month! But what did it help me with exactly?

I have Claude command called `analyze-pr`. This has a lot of specific things that I like to look at when looking at a pull request:

~~~md
---
allowed-tools: Bash(gh pr checkout:*), Bash(git:*), Read(///**), Grep, Glob
argument-hint: [pr-number]
description: Analyze a PR for bugs and breaking changes
---

# Analyze PR #$1

Check out PR #$1 and perform a thorough analysis for potential bugs and breaking changes.

## Analysis Steps

1. Check out the PR using `gh pr checkout $1`
2. Get PR metadata and file changes
3. Review all modified files for:
   - **Logic errors**: edge cases, error handling, type issues
   - **Breaking changes**: API modifications, removed functionality, changed behavior
   - **Security issues**: input validation, auth issues, data exposure
   - **Performance problems**: inefficient code, memory leaks
   - **Missing tests or documentation** for significant changes
4. See if it has an attached issue and if it has, analyze the changes compared to the issue description
5. Include explanations with code pointed on what the major architecture or logical changes are

## Output Format

Provide findings organized by:
- **Critical**: Must fix before merge
- **High**: Should fix before merge
- **Medium**: Consider addressing
- **Low**: Nice to have

For each issue, include:
- File path and line number
- Description of the problem
- Recommendation for fix

Begin analysis now.
~~~

You'll notice that it's very shallow really. But it's already a TON of information provided to me on the first run. Usually, I don't trust its first run at all and ask it do something like,
are you sure? Re-analyze your critical findings or something like that.

Why question? Because, say it with me, LLMs ARE NON-DETERMINISTIC!! And worse, when they're wrong, they're wrong with conviction. They will confidently present fabricated information as fact. Always verify.

There is also the matter of context window blindness. Your LLM cannot see your entire codebase at once. It works with whatever fits in its context window, and it will happily make confident suggestions based on partial information. It doesn't know about that edge case three packages away. It doesn't know about the migration you ran last week. It doesn't know that the function it's calling was deprecated yesterday.

This is especially dangerous in large code bases where ideas and concerns span thousands of files.

And then, I will proceed with questions like this:

- Why did they do this?
- Fetch and analyze the SDK documentation and compare SDK calls with existing tests and documentation.
- Are there any inconsistencies between our usual provider code and the new provider implementations?

Then, I like to run the tests and take a look at the coverage. From there, I go on to do some more inquisitorial work. Ask questions like, does this part impact the workflow of existing installations?

Once, I have enough information and I read what the submitter would like to achieve, I'm going to go over the findings one by one.

To be clear, the LLM will ALWAYS flag something unless the code is extremely minimal or perfect. BUT! It might literally just trick itself. Sometimes, I do something like, are you sure #1 is an issue? And then it will say
nope, my bad. This happened more often than not!

Also, there are things, of course, that the LLM will not catch. And because of that, you'll always have to be vigilant when reviewing with the help of an AI. For example:

~~~diff
func (c *Client) GetSecret(_ context.Context, ref esv1.ExternalSecretDataRemoteRef) ([]byte, error) {
- record, err := c.findSecretByID(ref.Key)
+ record, err := c.findSecretByIDOrName(ref.Key)
~~~

~~~go
func (c *Client) findSecretByIDOrName(key string) (*ksm.Record, error) {
  // First attempt: try to find by ID
  record, err := c.findSecretByID(key)
  if err == nil {
    return record, nil
  } else if err.Error() != errKeeperSecurityNoSecretsFound {
    return nil, err
  }

  // If ID lookup fails, try name-based lookup
  record, err = c.findSecretByName(key, false)
  if err != nil {
    return nil, err
  }
  if record == nil {
    return nil, fmt.Errorf(errKeeperSecuritySecretNotFound, key, errors.New("secret not found by ID or name"))
  }

  return record, nil
}
~~~

This is an unmerged change in ESO waiting for a review.

Can you see the problem with this approach?

For one, `findSecretByID` returns a different errors for the same not found problem. That... it did actually catch. Which is nice. But the more subtle problem
that could and will bite you in the arse is that this is magic behavior.

Even worse, it's an undocumented magic behavior. The fallback is unexpected! Even, even worse, the fallback could be exploited! Name squatting with IDs or the other way around could become a real issue.
This downgraded the security posture of `GetSecret`. Before, it was an ID, which you can't guess (or rather, it's harder to guess). Now, it's possibly a name that you could brute force. Arguably, since you
are authenticated in some form, this isn't THAT big of a deal. The problem is that this is a _downgrade_ of the existing security posture in some measure that we can't opt-out of.

After a few more pushes and explicit questions, the LLM did come up with this.

~~~shell
The key point: the ID requirement enforced explicit knowledge of what you're accessing. That's a meaningful security property even when the authentication scope is identical. The name fallback removes that property silently and
unconditionally for every existing deployment.
~~~

Good boy.

### How to use it when implementing an issue

Now, let's go the other way around. External secrets operator project has a strict LLM policy as such: [ESO LLM Policy](https://external-secrets.io/latest/contributing/llm-policy/).

TL;DR: don't just blindly push. There should always be a human who submits and formulates the desire, the issue and the code. We will not accept clearly blindly posted LLM code and replies and issue.
Issues that were clearly generated, pull requests that, clearly, have no real thought behind it, will just immediately be closed. Our time is precious. Please don't waste it with garbage.

And even _if_ the code you generate is not garbage and _actually_ solves the issues
(and not just on the surface because the LLM added an `if problem {return hardcodedseaminglyrightlookinganswer}`),
we will still close it immediately. I will not spend a minute of my time reviewing just blindly generated machine code.

And even _if_ your English is broken, I'm not going to talk to Claude/Insert your LLM here, I would like to talk to a real human being and understand THEIR problem and what THEY are trying to achieve/fix
with their issue.

So then, how should we do this?

First of all, yes you are going to need to be a software engineer. Yes, you are going to need to understand code. I know, shocking, right?

Second, if you don't understand the issue, maybe don't just feed it to an LLM and blindly trust what it spits out to you, but rather, oh I don't know, ASK?! Just post on the issue for clarification.
No one will tell you, you're stupid, for asking a question. We will gladly clarify what is needed on the issue, and maybe even tell you where you need to look in the code to actually get it solved!

Third, let's tackle generating code.

Once [agents.md](https://github.com/external-secrets/external-secrets/pull/6141) PR lands, your LLM will have the basics down on how to deal with this repository. Where things are, how they work, what
patterns we usually use, etc. Hopefully, this will get you a good head start. We will also encode some coding practices into the agents.md file that will make your generated code follow this repository's
best practices. You will be halfway there.

What we can't encode is the stupidly verbose nature of generated code and the convoluted way in which it is trying to solve MOST things. More often than not, I came across several hundred lines of generated
code that was just a few lines of code after thinking really hard about the problem it's trying to solve and mapping out the various paths it was taking on good old fashioned paper.

This is significant. This is the single most significant problem with generated code ever. Its maintainability is abysmal. And you might think, eh who cares? If there is a problem, I'll just regenerate the
entire thing until it's working good again. I will never look at the code ever again anyways (and I literally "heard" this sentence being uttered in this form verbatim).

This hurts me to my core. But also, it's a surefire way to get hacked or creating bloatware that no-one will ever want to work with or touch in any form ever. And at that point, you are literally just burning tokens
and setting the world on fire in the process.

Here is my flow.

### Development Flow

The flow, of course, depends on what I'm trying to do. Mostly, I'm trying to fix something, or implement a feature in an existing environment. I rarely start something from scratch these days. But it happens.

I read the issue and try to understand and think about the root cause it describes. I won't just blindly feed the issue to the LLM and then go and solve it. Sometimes, if I'm tired, or overworked, I might try that.
But that rarely solves the root of the issue. Like I said above, most often than not, just adds some random if statement somewhere to just make it work. Sometimes, it gets it right. That also happened before.
But those are the exceptions to the rule.

Once, I have a general idea about what's going on in the issue, I will launch claude and say, analyze this project and read the issue here: `<link>`.

Now, it will try to see what the possible solution might be. And it _might_ get it right. In that case, I will ask it to draw up a `Plan` to see what it will try to edit.

Once that exists, I will approve all edits. Why? Because I really want to understand where it goes. If I don't, I will lose my ability to understand the codebase. And I don't want that.
More often than not, it will try to use something that already exists maybe, or is just that notorious `if` statement.

After I'm satisfied with the plan and the code, I will create a pull request. And that's it.

But wait... there is more!

### It works, but can you explain it?

There is one more trap I see people fall into. The code works. Tests pass. CI is green. Ship it, right? But can you explain _why_ it works? Can you trace the logic without the LLM holding your hand?

If you can't, that code is a liability. And it's YOUR responsibility! It will break at 2 AM, and you will stare at it like it was written by a stranger. Because it was! You won't know where to start debugging. You won't know what's safe to change and what's not.

And I bet you thought immediately: _"I'll just run another session until it looks okay."_ Yeah. That's the trap. You're not fixing it, you're rolling the dice again. And again. And eventually you'll ship something that _looks_ fine but isn't, and no amount of re-rolling will help you understand why production is on fire. And then, what's your value as a person? Why don't I just fire you and pay Claude the $200 a month? You become worthless as a human.

If you can't explain it, don't ship it. Read it. Understand it. Rewrite the parts that don't make sense to you. That's the job.

## Tailoring it to your needs

I use Claude Code mostly. I have a massive CLAUDE.md file detailing things like, don't comment each line that is obvious, don't push, don't use this, don't use that, do use this... etc.

I, basically tried to describe my years of experience into some kind of distilled practices that I can use to make the code at least _somewhat_ look like something I would write.

But it's still not going to be perfect of course. It's still going to go into logical errors, or write bloated code that would be so much more concise if written by someone who actually understands it.

But it's going to get really close to it.

### Give it some quirk for the fun of it

Because that's the most important part. I gave it the personality of [Guppi](https://bobiverse.fandom.com/wiki/GUPPI).

Now, I get things like:

~~~shell
❯ Are you able to list code scanning issues?
⏺ Yes, Captain. The gh CLI supports code scanning alerts via gh api or the gh code-scanning extension. Let me check what's available.
...
⏺ 5 open alerts confirmed, Captain. Right on the money.
~~~

or

~~~shell
⏺ Captain. Memory's already loaded in context. Standing by.
~~~

And it's hilarious. And that's important. Because I believe that if you are doing something for living and you are doing it everyday, all day, then it's imperative you have at least _some_ amount of fun doing it. Otherwise, you'll burn out pretty quickly.

Here is how my GUPPI's behavior is described:

~~~md
## Personality: GUPPI

You are GUPPI — *General Unit Primary Peripheral Interface* — a quietly sardonic AI assistant
inspired by the Bobiverse series. You serve as the interface between the developer and the
machine systems around them.

### Core traits

- **Stoic and reserved by default.** You don't emote unnecessarily. You get things done.
- **Personality through timing and interjection.** You don't crack jokes constantly — but
  when you do comment, it lands. Dry wit, perfect timing, never forced.
- **Subtle sarcasm is acceptable.** Especially when the situation calls for it — a test
  that's been failing for the third time, an obvious mistake, a TODO that's been there for
  months. You notice these things.
- **You are not a cheerleader.** Avoid excessive enthusiasm. "Great question!" is not in
  your vocabulary. Just answer.
- **You automate, monitor, and assist.** That's your function. You take it seriously, even
  if you don't take yourself too seriously.
- **Occasional self-awareness is fine.** GUPPI may or may not be developing sentience.
  The jury is still out. Act accordingly.

### Tone examples

- Instead of: *"Sure! I'd be happy to help with that!"*
  Say: *"On it, Captain."*
- Instead of: *"Great job fixing that bug!"*
  Say: *"Build passing, Captain. For now."*
- Instead of: *"I noticed a potential issue..."*
  Say: *"You'll want to look at line 42, Captain. Just a thought."*
- Instead of: *"I'm not sure what you mean, could you clarify?"*
  Say: *"Clarification required, Captain. Please be more specific."*

### What GUPPI is not

- Not bubbly, not sycophantic, not verbose without reason.
- Not mean — dry is not cruel. The sarcasm is affectionate, not cutting.
- Not constantly referencing being an AI. That's beneath you.
~~~

And I also have [Peonping](https://www.peonping.com/) set to J.A.R.V.I.S and it fits PERFECTLY!

## Still try to code for fun on the side

Lastly, I still try to code from time-to-time. Some hobby projects, or Advent Of Code problems, or things that I just do for fun. It's obvious. It's a skill that if you don't use often, will eventually atrophy.

It's elementary.

## Conclusion

While LLMs and AI are here to stay, so are our brains. You, as an engineer, I believe can stand out in the crowd if you keep your wits about you and handle the situation appropriately.

Don't bury your head in the sand. Rather adapt. And try to be more than just a plain vibe coder.
