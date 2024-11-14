+++
author = "hannibal"
categories = ["Go", "Programming", "Performance"]
date = "2016-01-05"
title = "Improving performance with byte slice and int map"
url = "/2016/01/05/improving-performance-with-byte-slice-and-int-map"

+++

Hello Folks.

Today I would like to share with you my little tale of refactoring my solution to [Advent Of Code Day 13](http://adventofcode.com/day/13).

It's a lovely tale of action, adventure, drama, and comedy.

Let's being with my first iteration of the problem.

~~~go
package main

import (
	"bufio"
	"fmt"
	"math"
	"os"
	"strconv"
	"strings"

	"github.com/skarlso/goutils/arrayutils"
)

var seatingCombinations = make([][]string, 0)
var table = make(map[string][]map[string]int)
var keys = make([]string, 0)

//Person a person
type Person struct {
	// neighbour *Person
	name string
	like int
}

func main() {
	file, _ := os.Open("input.txt")
	defer file.Close()
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		line := scanner.Text()
		split := strings.Split(line, " ")
		like, _ := strconv.Atoi(split[3]) //If lose -> * -1
		if split[2] == "lose" {
			like *= -1
		}
		table[split[0]] = append(table[split[0]], map[string]int{strings.Trim(split[10], "."): like})
		if !arrayutils.ContainsString(keys, split[0]) {
			keys = append(keys, split[0])
		}
	}
	generatePermutation(keys, len(keys))
	fmt.Println("Best seating efficiency:", calculateSeatingEfficiancy())
}

func generatePermutation(s []string, n int) {
	if n == 1 {
		news := make([]string, len(s))
		copy(news, s)
		seatingCombinations = append(seatingCombinations, news)
	}
	for i := 0; i < n; i++ {
		s[i], s[n-1] = s[n-1], s[i]
		generatePermutation(s, n-1)
		s[i], s[n-1] = s[n-1], s[i]
	}
}

func calculateSeatingEfficiancy() int {
	bestSeating := math.MinInt64
	for _, v := range seatingCombinations {
		calculatedOrder := 0

		for i := range v {
			left := (i - 1) % len(v)
			//This is to work around the fact that in Go
			//modulo of a negative number will not return a positive number.
			//So -1 % 4 will not return 3 but -1. In that case we add length.
			if left < 0 {
				left += len(v)
			}
			right := (i + 1) % len(v)
			// fmt.Printf("Left: %d; Right: %d\n", left, right)
			leftLike := getLikeForTargetConnect(v[i], v[left])
			rightLike := getLikeForTargetConnect(v[i], v[right])
			// fmt.Printf("Name: %s; Left:%d; Right:%d\n", v[i], leftLike, rightLike)
			calculatedOrder += leftLike + rightLike
		}
		// fmt.Printf("Order for: %v; Calc:%d\n", v, calculatedOrder)
		if calculatedOrder > bestSeating {
			bestSeating = calculatedOrder
		}
	}

	return bestSeating
}

func getLikeForTargetConnect(name string, neighbour string) int {
	neighbours := table[name]
	for _, t := range neighbours {
		if v, ok := t[neighbour]; ok {
			return v
		}
	}
	return 0
}
~~~

This is quiet large. And takes a bit of explaining. So what is happening here? We are putting the names which correspond with numbers and neighbours into a map which has a map as a value. The map contains seating information for a person. For example, next to Alice, a bunch of people can sit, and they have a certain relationship to Alice, represented by a number.

We could, at this point, represent it with a graph, but that would be overkill.

Permutation is simple because I choose to represent a Table with a Circular Slice. This means that a slice like this => Alice, Bob, Tom; means that Alice is sitting next to Bob and Tom. So Alice's neighbour of -1 (left) is in fact i-1 % 3. And Bob is i + 1. For Tom, Alice is i + 1 % 3. After we got this, we just permutate the possible combinations into slices of slices and iterate over them.

The benchmark for this is terrible.

~~~bash

================With Strings================
20	 589571259 ns/op
ok  	github.com/skarlso/goprojects/advent/day13/part1	11.873s
~~~

So, my first thought was, convert everything I can to []byte. But because slices cannot be map keys, because map keys need to be comparable, we are still stuck with the same ns/ops.

~~~go
//var seatingCombinations = make([][]string, 0)
//var keys = make([]string, 0)
var seatingCombinations = make([][][]byte, 0)
var keys = make([][]byte, 0)
~~~

And I adjusted the code to work with []byte instead. What can we do to fix the map though? One obvious gain is, not to use string as a key. Because strings are immutable, working with them always means copy-ing and that's why they get to be very slow. So removing them from Keys and using Numbers instead will mean a huge gain for us.

To do this, I created a map which maps names with numbers. I could hardcode them with iota, but that is a very bad thing to do. It would mean, that when I add a new name, I would have to go, and re-compile my code, because data changed. That's not what we want.

So, I added this little tid-bit into the for cycle when I'm reading in the file lines =>

~~~go
...
if _, ok := nameMapping[split[0]]; !ok {
    nameMapping[split[0]] = id
    id++
}
if _, ok := nameMapping[trimmedNeighbour]; !ok {
    nameMapping[trimmedNeighbour] = id
    id++
}
...
~~~

Id starts as Zero. And nameMapping is a simple map[string]int. After this, we fix all the map calls, from ```table[split[0]]``` to ```table[nameMapping[split[0]]]```. Table's map will now work with int, but we can still work with strings otherwise.

~~~go
table[nameMapping[split[0]]] = append(table[nameMapping[split[0]]], map[int]int{nameMapping[trimmedNeighbour]: like})
~~~

This has now a marginally better performance as before:

~~~bash
BenchmarkCalculateSeating	      50	  32637879 ns/op
ok  	github.com/skarlso/goprojects/advent/day13/part1	1.698s
~~~

But, we can still do a HUGE one better. Can you notice the other bottleneck? See, how keys are still []byte? That's, now completely unnecessary. We can use int, since our keys are ints! _Permutation_ changes, and the retrieve.

~~~go
...
func generatePermutation(s []int, n int) {
...

...
func getLikeForTargetConnect(name int, neighbour int) int {
	neighbours := table[name]
	for _, t := range neighbours {
		if v, ok := t[neighbour]; ok {
			return v
		}
	}
	return 0
}
...
~~~

Permutation was the Other huge performance consumption. Now, our run time is.... drum rolls...

~~~bash
BenchmarkCalculateSeating	   10000	    166431 ns/op
ok  	github.com/skarlso/goprojects/advent/day13/part1	1.695s
~~~

Down to 166431 ns/op!!! From 32637879 ns/op!! And notice how suddenly, go's benchmark jumped up in sample count. Our code is now blazing fast. It's 0.05% of the previous run! It's almost **200 times faster**!

We could still improve it here and there. I'm sure I'm doing some extra stuff which is not needed or could be made easier somehow. But I'm actually quiet happy with this solution right now.

The full code:

~~~go
package main

import (
	"bufio"
	"math"
	"os"
	"strconv"
	"strings"

	"github.com/skarlso/goutils/arrayutils"
)

var seatingCombinations = make([][]int, 0)
var table = make(map[int][]map[int]int)
var keys = make([]int, 0)
var nameMapping = make(map[string]int)

//Person a person
type Person struct {
	// neighbour *Person
	name string
	like int
}

func main() {
	CalculatePerfectSeating()
}

//CalculatePerfectSeating returns the perfect seating order based on Love/Hate relations
func CalculatePerfectSeating() {
	file, _ := os.Open("input.txt")
	defer file.Close()
	scanner := bufio.NewScanner(file)
	id := 0
	for scanner.Scan() {
		line := scanner.Text()
		split := strings.Split(line, " ")
		trimmedNeighbour := strings.Trim(split[10], ".")
		like, _ := strconv.Atoi(split[3]) //If lose -> * -1
		if _, ok := nameMapping[split[0]]; !ok {
			nameMapping[split[0]] = id
			id++
		}
		if _, ok := nameMapping[trimmedNeighbour]; !ok {
			nameMapping[trimmedNeighbour] = id
			id++
		}
		if split[2] == "lose" {
			like *= -1
		}
		table[nameMapping[split[0]]] = append(table[nameMapping[split[0]]], map[int]int{nameMapping[trimmedNeighbour]: like})
		if !arrayutils.ContainsInt(keys, nameMapping[split[0]]) {
			keys = append(keys, nameMapping[split[0]])
		}
	}
	generatePermutation(keys, len(keys))
	// fmt.Println("Best seating efficiency:", calculateSeatingEfficiancy())
}

func generatePermutation(s []int, n int) {
	if n == 1 {
		news := make([]int, len(s))
		copy(news, s)
		seatingCombinations = append(seatingCombinations, news)
	}
	for i := 0; i < n; i++ {
		s[i], s[n-1] = s[n-1], s[i]
		generatePermutation(s, n-1)
		s[i], s[n-1] = s[n-1], s[i]
	}
}

func calculateSeatingEfficiancy() int {
	bestSeating := math.MinInt64
	for _, v := range seatingCombinations {
		calculatedOrder := 0

		for i := range v {
			left := (i - 1) % len(v)
			//This is to work around the fact that in Go
			//modulo of a negative number will not return a positive number.
			//So -1 % 4 will not return 3 but -1. In that case we add length.
			if left < 0 {
				left += len(v)
			}
			right := (i + 1) % len(v)
			calculatedOrder += getLikeForTargetConnect(v[i], v[left]) + getLikeForTargetConnect(v[i], v[right])
		}
		if calculatedOrder > bestSeating {
			bestSeating = calculatedOrder
		}
	}

	return bestSeating
}

func getLikeForTargetConnect(name int, neighbour int) int {
	neighbours := table[name]
	for _, t := range neighbours {
		if v, ok := t[neighbour]; ok {
			return v
		}
	}
	return 0
}
~~~

Also, on github => [Advent Of Code Day 13](https://github.com/Skarlso/goprojects/tree/master/advent/day13).

Thank you very much for reading, this has been a massive fun to write and to refactor.

Have something to say? Please don't hesitate.

And as always,

Have a nice day!

Gergely.
