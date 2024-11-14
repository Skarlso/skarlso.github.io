+++
author = "hannibal"
categories = ["aoc"]
date = "2021-12-17T01:01:00+01:00"
title = "Advent Of Code - Day 12 - Updated"
url = "/2021/12/17/aoc-day12-updated"
comments = true
+++

# Advent Of Code - Day 12 - Updated

A comment from one of my readers prompted me to revise my solution on this trying to do part 2. The suggestions was that
instead of using a `struct` as seen, use `int` and keep track of the count for small caves that way. I started to do
that but got into various problems along the way when I got frustrated with my code, deleted the whole thing and begun
again. But this lead me to a small, and better code than before which actually worked.

## Day 12 - Part 1

There are a number of things I realized and improved upon. Let's take them one by one.

### Parsing

I did this whole convoluted way of parsing the "graph" into an actual graph when I saw two things. First, I don't need a
graph I just need the values. Second, if you don't need the graph, you don't have anything to initialize and you can use
Go map's feature of creating default entries for default keys.

Go maps initialize to value default. For reference types that's nil. For primitives, whatever the primitives zero value
is. Empty string, `0`, etc.

This lead me to parsing the input such as:

```go
	input := strings.Split(string(content), "\n")
	caves := make(map[string][]string)
	for _, line := range input {
		var (
			a, b string
		)
		split := strings.Split(line, "-")
		a = split[0]
		b = split[1]
		caves[a] = append(caves[a], b)
		caves[b] = append(caves[b], a)
	}
```

And that's it. `append` deals with `nil` slices and creates the slice if needed. Meaning if the entry doesn't exist,
append will create a new slice for it.

### State

I need to keep state of the caves I'm visiting somehow. After reading a bunch of more information on DFS and BFS and
listing all paths and all those things I never read otherwise every in my day to day life, I saw, that I for part two,
and in fact, for part 1 as well, I need a state which I can refresh based on the current location.

Previously, my recursive dfs was doing that with the `path` passed in. That's all fine, but I also need now if I ever
visited a cave twice. The suggestions of using an int for visited is nice, but somehow, wasn't working for me, because I
kept messing up the subtraction when recursing back. So I decided that I drop the recursing and try to do this
iteratively.

Second, I also saw in most of the cases I don't care about the path, I just need a count. I just kept the path for debug
to see where my algorithm is going. So I ditched that and just increase a counter.

To keep the state I'm using a `struct` which will have the current value of the cave, the path so far which was
travelled from the current location and so far and the fact if I visited this cave before.

```go
type cave struct {
	value     string
	pathSoFar []string
	twice     bool
}
```

This allows for a number of things. First, I don't need to pass around path recursively, I will have a reference to it
at the current visited location. Second, I can track the actual state in the cave, like paint the cave if I already saw
that one. Once that happens, every other cave will have this value passed around, meaning all other caves will come in
with a flag, that hey, I already visited a cave twice before, so I'm not gonna do that again.

### The Loop

No, not the amazon prime series The Loop ( which is fantastic btw ), but the main for loop of this logic. Now that I
have my state, I can just do a BFS on this baby. The main loop changes slightly. We now, keep track of caves, not just
the values.

But it's super easy. Barely an inconvenience. Oh really? Yeah, you see, we just make our queue have caves. But what
about the `seen` part? This changes as well. Now, because we pass around the path taken so far, we'll have to look in
that path if the neighbor if our current cave is already in the path that has been taken so far for that save.

Just create a small `Contains` method ( which will be obsolete after finally go 1.18 comes out with generics and we'll
have the mighty `Filter` or `Search` on our side!!!!!! ) like this:

```go
	seen := func(a string, b []string) bool {
		for _, v := range b {
			if v == a {
				return true
			}
		}
		return false
	}
```

... and then we can look in the path if we met this cave before. What I also realized was that I was turning the cave
value into lowercase and checked it that way, when I could have just said that if it matched with the ToLower version
of the name it is lower case. :facepalm:. But this is why I'm doing AOC. To learn and evolve. Hopefully, I learned a lot
from this day.

So the main loop:

```go
	count := 0
	start := cave{
		pos:       "start",
		pathSoFar: []string{"start"},
	}
	queue := []cave{start}
	var current cave
	for len(queue) > 0 {
		current, queue = queue[0], queue[1:]
		if current.pos == "end" {
			count++
			continue
		}
		for _, next := range caves[current.pos] {
			if !seen(next, current.pathSoFar) {
				path := make([]string, 0)
				path = append(path, current.pathSoFar...)
				if strings.ToLower(next) == next {
					path = append(path, next)
				}
				queue = append(queue, cave{
					pos:       next,
					pathSoFar: path,
				})
			}
		}
	}
```

And that's it. We print `count` and we have our number.

## Day 12 - Part 2

Now that we have our state in the loop itself, we just add a new variable to the `cave` called `twice` (it was already
there but I haven't used it yet). Our criteria for twice is as follows. If we haven't seen it yet, we explore it. If it
is already in the path so far, from which we came in AND none of the caves so far were visited twice ( which we'll know
because we bring our marker with us during the loop ) we set our cave to visited ( paint it ) and we only take the path
so far with us, so we don't put this guy in the path YET. We'll eventually do that though on the next time this cave is
encountered at which point it will bring twice with it.

How does that look like in code?

```go
	start := cave{
		value:     "start",
		pathSoFar: []string{"start"},
	}
	queue := []cave{start}
	var current cave
	for len(queue) > 0 {
		current, queue = queue[0], queue[1:]
		if current.value == "end" {
			count++
			continue
		}
		for _, next := range caves[current.value] {
			seenThisCaveBefore := seen(next, current.pathSoFar)
			if !seenThisCaveBefore {
				path := make([]string, 0)
				path = append(path, current.pathSoFar...)
				if strings.ToLower(next) == next {
					path = append(path, next)
				}
				queue = append(queue, cave{
					value:     next,
					pathSoFar: path,
					twice:     current.twice,
				})
			} else if seenThisCaveBefore && !current.twice && !seen(next, []string{"start", "end"}) {
				queue = append(queue, cave{
					value:     next,
					pathSoFar: current.pathSoFar,
					twice:     true,
				})
			}
		}
	}
```

Again, we start off by adding `start`. Oh, and we don't do that for `start` or `end`. Also realized that I can use
`seen` now as sort of like a check for a value NOT being one of two values instead of a convoluted if structure.

So if everything is as before, expect we also take `twice` with us, and if we've `seenThisCaveBefore` but we haven't
visited any caves twice before ( we take the marker with us during the journey ) and it's not start or end, we mark it
as visited twice, and we add pathSoFar as the current path so far, without adding this cave into it.

And that's it!

## Conclusion

This day was somehow difficult on the first run. I couldn't figure out a nice way to keep track of my journey so far.
What helped me eventually was the prompt from my reading and reading various articles about keeping track during a
BFS and how listing all paths in a graph work. Which can be a difficult problem especially when there is a circle in the
graph. This would probably not work in that case, but we are confident that Eric doesn't do tricks like that.

The repository for all my solutions for AOC 2021 can be found [here](https://github.com/Skarlso/aoc2021).

Thank you for reading!

Gergely.
