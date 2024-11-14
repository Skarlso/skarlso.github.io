+++
author = "hannibal"
categories = ["Go"]
date = "2016-08-19"
title = "Always Go with []byte"
url = "/2016/08/19/always-go-with-bytes"

+++

*Update*: This post ignored the fact that this works for utf-8 characters only. Characters which are stored on more than 1 byte
will cause trouble. Look at this [Effective Go Example](https://go.dev/doc/effective_go#for).

~~~go
for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
~~~

Prints:

~~~
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
~~~

Keep this in mind when working with strings.

Another quick reminder... Always go with []byte if possible. I said it before, and I'm going to say it over and over again. It's crucial.

Here is a little code from exercism.io. First, with strings:

~~~go
package igpay

import (
    "strings"
)

// PigLatin translates reguler old English into awesome pig-latin.
func PigLatin(in string) (ret string) {
    for _, v := range strings.Fields(in) {
        ret += pigLatin(v) + " "
    }

    return strings.Trim(ret, " ")
}

func pigLatin(in string) (ret string) {
    if strings.IndexAny(in, "aeiou") == 0 {
        ret += in + "ay"
        return
    }

    for i := 0; i < len(in); i++ {
        vowelPos := strings.IndexAny(in, "aeiou")

        if (in[0] == 'y' || in[0] == 'x') && vowelPos > 1 {
            vowelPos = 0
            ret = in
        }
        if vowelPos != 0 {
            adjustPosition := vowelPos

            if in[adjustPosition] == 'u' && in[adjustPosition - 1] == 'q' {
                adjustPosition++
            }

            ret = in[adjustPosition:] + in[:adjustPosition]
        }
    }
    ret += "ay"
    return
}
~~~

Than with []byte:

~~~go
package igpay

import (
    // "fmt"
    "bytes"
)

// PigLatin translates reguler old English into awesome pig-latin.
func PigLatin(in string) (ret string) {
    inBytes := []byte(in)
    var retBytes [][]byte
    for _, v := range bytes.Fields(inBytes) {
        v2 := make([]byte, len(v))
        copy(v2, v)
        retBytes = append(retBytes, pigLatin(v2))
    }

    ret = string(bytes.Join(retBytes, []byte(" ")))
    return
}

func pigLatin(in []byte) (ret []byte) {
    if bytes.IndexAny(in, "aeiou") == 0 {
        ret = append(in, []byte("ay")...)
        return
    }

    for i := 0; i < len(in); i++ {
        vowelPos := bytes.IndexAny(in, "aeiou")

        if (in[0] == 'y' || in[0] == 'x') && vowelPos > 1 {
            vowelPos = 0
            ret = in
        }
        if vowelPos != 0 {
            adjustPosition := vowelPos

            if in[adjustPosition] == 'u' && in[adjustPosition - 1] == 'q' {
                adjustPosition++
            }

            in = append(in[adjustPosition:], in[:adjustPosition]...)
            ret = in
            // fmt.Printf("%s\n", ret)
        }
    }
    ret = append(ret, []byte("ay")...)
    return
}
~~~

And than,the benchmarks of course:

~~~bash
BenchmarkPigLatin-8          	  200000	     10688 ns/op
BenchmarkPigLatinStrings-8   	  100000	     15211 ns/op
PASS
~~~

The improvement is not massive in this case, but it's more than enough to matter. And in a bigger, more complicated program, string concatenation will take a LOT of time away.

In Go, the `bytes` package has a 1-1 map compared to the `strings` packages, so chances are, if you are doing strings concatenations you will be able to port that piece of code easily to []byte.

That's all folks.

Happy coding,
Gergely.
