+++
author = "hannibal"
categories = ["problem solving", "programming"]
date = "2015-07-19"
tags = ["golang"]
title = "Converting numbers into string representations"
url = "/2015/07/19/converting-numbers-into-string-representations/"

+++

I quiet like this one. My first go program snippet without any peaking or googling. I'm proud, though it could be improved with a bit of struct magic and such and such. And it only counts 'till 1000.

~~~go

package main

import "fmt"

var words = map[int]string{1: "one", 2: "two", 3: "three", 4: "four", 5: "five", 6: "six", 7: "seven",
	8: "eight", 9: "nine", 10: "ten", 11: "eleven", 12: "twelve", 13: "thirteen", 14: "fourteen", 15: "fifteen",
	16: "sixteen", 17: "seventeen", 18: "eighteen", 19: "nineteen", 20: "twenty", 30: "thirty", 40: "forty",
	50: "fifty", 60: "sixty", 70: "seventy", 80: "eighty", 90: "ninety"}

// CountLetters count the letters in a long string number representation
func CountLetters(limit int) {
	myLongNumberString := ""
	for i := 1; i <= limit; i++ {
		addLettersToMyString(&myLongNumberString, i)
	}
	// fmt.Println("1-9 written with letters is: ", len(myLongNumberString))
	fmt.Println("The string is:", myLongNumberString)
	fmt.Println("Lenght of string is:", len(myLongNumberString))
}

func addLettersToMyString(myString *string, num int) {
	if num < 20 {
		*myString += words[num]
	}

	if num >= 20 && num < 100 {
		*myString += countMiddle(num)
	}

	if num >= 100 && num < 1000 {
		hundred, tenth := countHundred(num)
		if tenth ==  {
			*myString += hundred
		} else if tenth >= 11 && tenth < 20 {
			*myString += hundred + "and" + words[tenth]
		} else {
			*myString += hundred + "and" + countMiddle(tenth)
		}

	}

	if num == 1000 {
		*myString += "onethousand"
	}
}

func countMiddle(num int) string {
	minues := num % 10
	num -= minues
	return words[num] + words[minues]
}

func countHundred(num int) (string, int) {
	minues := num % 100
	num -= minues
	return (words[(num/100)] + "hundred"), minues
}
~~~