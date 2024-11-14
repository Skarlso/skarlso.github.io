+++
author = "hannibal"
categories = ["programming"]
date = "2015-07-15"
tags = ["golang"]
title = "Bitwise & Operator"
url = "/2015/07/15/bitwise-operator/"

+++

The first, and only time so far, that I got to use the bitwise & operator. I enjoyed doing so!!

And of course from now on, I'll be looking for more opportunities to (ab)use it.

~~~go

package secret

import "sort"

const REVERSE = 16

func Handshake(code int) []string {
    // binary_rep := convertDecimalToBinary(code)
    if code <  { return nil }
    secret_map := map[int]string {
        1: "wink",
        2: "double blink",
        4: "close your eyes",
        8: "jump",
    }

    var keys []int
    for k := range secret_map {
        keys = append(keys, k)
    }
    // To make sure iteration is always in the same order.
    sort.Ints(keys)

    code_array := make([]string, )
    for _, key := range keys {
        if code & key == key {
            code_array = append(code_array, secret_map[key])
        }
    }

    if code & REVERSE == REVERSE {
        code_array = reverse_array(code_array)
    }

    return code_array
}

func reverse_array (array_to_reverse []string) []string {
    for i, j := , len(array_to_reverse) -1 ; i < j; i, j = i + 1, j - 1 {
        array_to_reverse[i], array_to_reverse[j] = array_to_reverse[j], array_to_reverse[i]
    }
    return array_to_reverse
}
~~~