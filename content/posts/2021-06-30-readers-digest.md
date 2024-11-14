+++
author = "hannibal"
categories = ["books"]
date = "2021-06-30T01:01:00+01:00"
title = "Reader's Digest 2021-06"
url = "/2021/06/30/readers-digest"
comments = true
+++

# Reader's Digest

I thought it would a cool idea if I kept a summary of the things I've read or listened to on a monthly
basis. Here is June of 2021 so far. Enjoy.

## Hell Divers 6: Allegiance

I love the Hell Divers series. I listened to this one on audible as always and it was fantastic, as always.
King Xavier Rodriguez. That sounds as bad ass as it is. We follow King Xavier and Rhino, his trusted subordinate into
battle with a new threat called Skin Walkers and of course the machines, the defectors which are trying to kill everyone.
Also, there is now a signal from a possible group of people who are surviving on the surface for more than 200 years!
Xavier hates being king but does anything to keep the peace and humanity alive. Sadly, not all of his people agree to that
especially not after what they did to their people. The people of the metal island, the cannibals are harsh people. And
they don't get along nicely all the time.

## Go Typed Parameters Proposal

The proposal can be found [here](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md) and it's quite lengthy.

The first example speaks worlds:

~~~go
// Print prints the elements of a slice.
// It should be possible to call this with any slice value.
func Print(s []T) { // Just an example, not the suggested syntax.
	for _, v := range s {
		fmt.Println(v)
	}
}
~~~

It is actually exciting that generics is coming to Go. And it's even in my life time. The introduction of generics back
in the old days to Java brought a lot of confusion into the land of Java but more importantly, it brought in the eagerness
of using generics for EVERYTHING. I'm really hoping that this proposal will not affect the land of Go. But unfortunately,
it probably will. It is up to us to take care and apply generics WHEN IT'S REALLY NECESSARY AND ONLY THEN!

Even so, the proposal is really solid and has some pretty nice scenarios. And there is this issue:
[Add generic slice package to Go](https://github.com/golang/go/issues/45955). Which talks about what the title says.
It has some really amazing feedback already and a gazillion view and an "approve" from Rob Pike! [Comment](https://github.com/golang/go/issues/45955#issuecomment-861173504).
Which just means he didn't vehemently object. This is really happening. Which is insane and exciting at the same time!

## Horizon Zero Dawn

This isn't a book, but a game on PS4 which I spent most of my time instead of reading. I do not regret it, because the
story is absolutely beautiful. When I play, I play for reading. Meaning I read every lore and listen to every story I
encounter and start looking for them as well. Now, let's step back, what is Horizon Zero Dawn?

If you've been living under a rock these past years, or are just not into games that much let me break it down for you.
The year is.... X. We have no idea but it's somewhere around the 31st century. It's been several decades after some kind
of apocalypse ( what kind, we find out as we progress with the story ) which left the world in ruins. Small tribes of
humanity survived and segregated into the Nora, Oseram, Banuk and Carja mostly. The ones we know about that is.
Our protagonist is a Nora girl called Aloy.

But there is an interesting twist... Instead of mostly animals, we have giant Robots, which are animals / perform the tasks
animals perform. Like, hunting, scavenging, eating plant life, swimming in the waters... Etc.

And these people, hunt these machines. Usually, these machines are docile, but with time, they got more and more aggressive
and more and more machines came around which were always aggressive and effectively started to hunt people.

**SPOILER WARNING**: Now, our girl grows up as an outcast, because she actually came into the world, not from a mother
but from the belly of a mountain called Mother's Heart. Mother's Heart is actually an ancient facility sealed shut, with
a biometric security system which they call All Mother. The story is REALLY long, but eventually, we find out the following...

A guy called Faro creates a company called Faro Automated Solutions which creates a bunch of semi-sentient machines which
are used as "peacekeeping" devices. They can self replicate and they "eat biomass" to sustain themselves indefinitely.
And of course, as these things are, there is a glitch and they become something called a Swarm which devourers everything
in their path. Until Dr. Elisabeth Sobeck comes along and saves the day with Project Zero Dawn.

And here comes the twist. Sobeck knows that the planet is doomed. There is a cryptographic thing where they need hundreds
of years to crack the machines encryption and shut them down. But humanity and anything that's a biomass doesn't have that
long. So she proposes to create Gaia. Which is a powerful AI that has various subroutine helpers called Hades, Apollo and
others. They contain the digitized version of everything living and after humanity and everything is gone, Gaia will
crack the code eventually in an underground shelter and repopulate the Earth with life. And that's where the robot animals
come in, they are basically terraforming. Making life possible.

But as these things go, something went wrong and Apollo, which contained all knowledge on Earth, has been destroyed and
the people reborn had no idea what's going on, where they are and how they are alive. The initial people were created
in tubes from zygotes.

And here comes Aloy. So Gaia had a subroutine called Hades, to destroy everything if terraforming failed or there was a
catastrophic event and the world needs resetting. Gaia can't do that because she is a fully featured, feeling, sentient
AI who is incapable of destruction and Hades is the opposite. It can take over, destroy, then give back control to Gaia.
Of course, Hades was corrupted and didn't want to give back control and started killing everything. Gaia created Aloy who
is a clone of Elisabeth Sobeck so she could access facilities with biometric scanners all over the place and find out the
truth about the world and fix Gaia and delete Hades.

It's quite the story! Incredibly well written with likable characters all along the way. It was well worth the time spent
on it, even if I didn't catch up on some reading. I regret nothing.

## Conclusion

That's it for these months. Next month I'm planning on finishing two Go related books hopefully and will write some reviews
about those.

And as always,
Thanks for reading,
Gergely.
