+++
author = "hannibal"
categories = ["Go", "Programming"]
date = "2015-12-29"
title = "Use Byte Array Instead of Strings"
url = "/2015/12/29/use-byte-array-instead-of-strings/"

+++

Hello Folks.

This is just a quick post on the topic and a reminder for myself and everybody to ALWAYS USE []BYTE INSTEAD OF STRINGS.

[]Byte is marginally faster than a simple Strings. In fact, I would say using []byte should be the standard instead of strings.

Sample code:

~~~go
package solutions

import "fmt"

const (
    //INPUT input
    INPUT = "1321131112"
    //LIMIT limit
    LIMIT = 50
)

//LookAndSay translates numbers according to Look and Say algo
func LookAndSay(s string, c chan string) {
    charCount := 1
    look := ""
    for i := range s {
        if i+1 < len(s) {
            if s[i] == s[i+1] {
                charCount++
            } else {
                look += fmt.Sprintf("%d%s", charCount, string(s[i]))
                charCount = 1
            }
        } else {
            look += fmt.Sprintf("%d%s", charCount, string(s[i]))
        }
    }
    c <- look
}

//GetLengthOfLookAndSay Retrieve the Length of a lookandsay done Limit times
func GetLengthOfLookAndSay() {
    c := make(chan string, 0)
    go LookAndSay(INPUT, c)
    finalString := <-c
    for i := 0; i <= LIMIT-2; i++ {
        go LookAndSay(finalString, c)
        finalString = <-c
        // fmt.Println(finalString)
    }
    fmt.Println("Lenght of final String:", len(finalString))
}

~~~

This, with the limit raised to 50 run for ~1 hour. Even with the routines although they were just for show since they had to wait for each others input.

Now change this to []byte and the run time was almost under 2 seconds on my machine.

~~~go
package solutions

import (
    "fmt"
    "strconv"
)

const (
    //LIMIT limit
    LIMIT = 50
)

//INPUT puzzle input
//This used to be a string until I was reminded that BYTE ARRAY IS ALWAYS FASTER!
var INPUT = []byte("1321131112")

//LookAndSay translates numbers according to Look and Say algo
func LookAndSay(s []byte) (look []byte) {
    charCount := 1
    for i := range s {
        if i+1 < len(s) {
            if s[i] == s[i+1] {
                charCount++
            } else {
                b := []byte(strconv.FormatInt(int64(charCount), 10))
                look = append(look, b[0], s[i])
                charCount = 1
            }
        } else {
            b := []byte(strconv.FormatInt(int64(charCount), 10))
            look = append(look, b[0], s[i])
        }
    }
    return
}

//GetLengthOfLookAndSay Retrieve the Length of a lookandsay done Limit times
func GetLengthOfLookAndSay() {
    finalString := INPUT
    for i := 0; i <= LIMIT-1; i++ {
        finalString = LookAndSay(finalString)
    }
    fmt.Println("Lenght of final String:", len(finalString))
}

~~~

This is the solution for Day 10 on [AdventOfCode](http://adventofcode.com/) by the way.

Thanks for readin'.
Gergely.
