+++
author = "hannibal"
categories = ["Go", "Programming", "Channels"]
date = "2016-01-01"
title = "Byte arrays and Channels"
url = "/2016/01/01/byte-arrays-and-channels"

+++

Hi folks and a Happy new Year!

Today, I would like to show you some interesting things you can do with channels. Consider the following simple example.

~~~go
package main

import "fmt"

func main() {
	generatedPassword := make(chan int, 100)
	correctPassword := make(chan int)
	defer close(generatedPassword)
	defer close(correctPassword)
	go passwordIncrement(generatedPassword)
	go checkPassword(generatedPassword, correctPassword)
	pass := <-correctPassword
	fmt.Println(pass)
}

func checkPassword(input <-chan int, output chan<- int) {
	for {
		p := <-input
		//Introduce lengthy operation here
		// time.Sleep(time.Second)
		fmt.Println("Checking p:", p)
		if p > 100000 {
			output <- p
		}
	}
}

func passwordIncrement(out chan<- int) {
	p := 0
	for {
		p++
		out <- p
	}
}
~~~

The premise is as follows. It launches two go routines. One, which generates passwords, and an other which checks for validity. The two routines talk to each other through the channel ```generatedPassword```. That's the providing connections between them. The channel ```correctPassword``` provides output for the ```checkPassword``` routine.

If there is data received from ```correctPassword``` channel, we found our first password and there is no need to look further so we, print the password and quit. The channels will close with defer. This works. But the password is usually either a []byte or a string. With string, it still works.

~~~go
package main

import "fmt"

func main() {
	generatedPassword := make(chan string, 100)
	correctPassword := make(chan string)
	defer close(generatedPassword)
	defer close(correctPassword)
	go passwordIncrement(generatedPassword)
	go checkPassword(generatedPassword, correctPassword)
	pass := <-correctPassword
	fmt.Println(pass)
}

func checkPassword(input <-chan string, output chan<- string) {
	for {
		p := <-input
		//Introduce lengthy operation here
		// time.Sleep(time.Second)
		fmt.Println("Checking p:", p)
		if performSomeCheckingOperation(p) {
			output <- p
		}
	}
}

func generateNewPassword(out chan<- string) {
	var p string
	for {
		p = generate(p)
		out <- p
	}
}
~~~

The generating happens based on the previously generated password. For example, we increment, or permeate. aaaa, aaab, aaac...

So ```generatedPassword``` is a buffered channel, it gathers a 100 passwords from which checking retrieves passwords one by one and works on them in a slower process.

Now, this is fine, but using []byte arrays will always be more powerful and faster. So we would like to use []byte. Like this:

~~~go
package main

import "fmt"

func main() {
	generatedPassword := make(chan []byte, 100)
	correctPassword := make(chan []byte)
	defer close(generatedPassword)
	defer close(correctPassword)
	go passwordIncrement(generatedPassword)
	go checkPassword(generatedPassword, correctPassword)
	pass := <-correctPassword
	fmt.Println(string(pass))
}

func checkPassword(input <-chan []byte, output chan<- []byte) {
	for {
		p := <-input
		//Introduce lengthy operation here
		// time.Sleep(time.Second)
		fmt.Println("Checking p:", string(p))
		if performSomeCheckingOperation(p) {
			output <- p
		}
	}
}

func generateNewPassword(out chan<- []byte) {
	var p []byte
	for {
		p = generate(p)
		out <- p
	}
}
~~~

This will not work. Why? Because []byte is a slice and thus will be constantly overwritten. The checking go routine will always only check the last data and many generated passwords will be lost. This is also noted in go's scanner here => [Scanner.Bytes](https://golang.org/pkg/bufio/#Scanner.Bytes)

We have a couple of options here.

We could use ```string``` channels and convert to []byte after. This is still okay, because the conversion isn't very CPU intensive.

~~~go
...
generatedPassword := make(chan string, 100)
correctPassword := make(chan string)
...
p := []byte(<-input) //This will work very nicely.
...
~~~

Options two would be If you have a fixed password to handle, fix data, for example MD5 hash, you can use a byte array. Like this:

~~~go
package main

import "fmt"

const PASSWD=13

func main() {
	generatedPassword := make(chan [PASSWD]byte, 100)
	correctPassword := make(chan [PASSWD]byte)
	defer close(generatedPassword)
	defer close(correctPassword)
	go passwordIncrement(generatedPassword)
	go checkPassword(generatedPassword, correctPassword)
	pass := <-correctPassword
	fmt.Println(string(pass))
}

func checkPassword(input <-chan [PASSWD]byte, output chan<- [PASSWD]byte) {
	for {
		p := <-input
		//Introduce lengthy operation here
		// time.Sleep(time.Second)
		fmt.Println("Checking p:", string(p))
		if performSomeCheckingOperation(p) {
			output <- p
		}
	}
}

func generateNewPassword(out chan<- [PASSWD]byte) {
	var p [PASSWD]byte
	for {
		p = generate(p)
		out <- p
	}
}
~~~

This is also one solution. If you have to convert between the two, could go with ```p := byte[:]```.

Conclusion is, that use conversion rather than string types and be aware that using slices in channels is dangerous.

Thanks for reading!
Gergely.
