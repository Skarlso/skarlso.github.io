+++
author = "hannibal"
categories = ["Go", "problem solving", "programming"]
date = "2015-07-30"
title = "Sieve of Eratosthenes in Go"
url = "/2015/07/30/sieve-of-eratosthenes-in-go/"

+++

I'm pretty proud of this one as well.

~~~go

package sieve

//Sieve Uses the Sieve of Eratosthenes to calculate primes to a certain limit
func Sieve(limit int) []int {
	var listOfPrimes []int
	markers := make([]bool, limit)

	for i := 2; i < limit; i++ {
		if !markers[i] {
			for j := i + i; j < limit; j += i {
				markers[j] = true
			}
			listOfPrimes = append(listOfPrimes, i)
		}
	}

	return listOfPrimes
}
~~~