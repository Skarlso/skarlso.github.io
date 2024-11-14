+++
author = "hannibal"
categories = ["Go", "Programming"]
date = "2015-12-23"
title = "Recursive Letter Frequency Count"
url = "/2015/12/23/recursive-freq-count/"

+++

Hello everybody!

I wanted to do a sort post about word frequency count. I did it many times now and I was curious as how a recursive solution would perform as opposed to looping.

So I wrote it up quickly and added a few benchmarks with different sized data.

First.... The code:

~~~go
var freqMap = make(map[string]int, 0)

func countLettersRecursive(s string) string {
    if len(s) == 0 {
        return s
    }
    freqMap[string(s[0])]++
    return countLettersRecursive(s[1:])
}

func countLettersLoop(s string) {
    for _, v := range s {
        freqMap[string(v)]++
    }
}
~~~

Very simple. The first run with a small sample: "asdfasdfasdfasdfasdf"

~~~bash
BenchmarkLoopFrequencyCount  5000000           377 ns/op
BenchmarkRecursiveFrequencyCount     5000000           380 ns/op
~~~

They almost equal but Recursive seems to be lagging behind. So I increased the sample size to a text which was 496 long.

~~~bash
PASS
BenchmarkLoopFrequencyCount    30000         53336 ns/op
BenchmarkRecursiveFrequencyCount       20000         61780 ns/op
~~~

And, as expected, recursing is less performant than looping. Also, I think my machine would die from a larger data size...

But the recursive looks so much cooler though.

Thanks for reading!
Gergely.
