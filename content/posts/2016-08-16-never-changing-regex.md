+++
author = "hannibal"
categories = ["Go"]
date = "2016-08-16"
title = "Global variable for never changing regex"
url = "/2016/08/16/never-changing-regex"

+++

Quick reminder. If you have a never changing regex in Go, do NOT put it into a frequently called function. ALWAYS put it into a global variable. I'll show you why.

Benchmark for code with a variable in a frequently called function:

~~~bash
BenchmarkNumber-8     	   30000	     41633 ns/op
BenchmarkAreaCode-8   	   50000	     27736 ns/op
BenchmarkFormat-8     	   50000	     29263 ns/op
PASS
ok  	_/phone-number	5.110s
~~~

Benchmark for code with the same variable outside in a global scope:

~~~bash
BenchmarkNumber-8     	  300000	      5618 ns/op
BenchmarkAreaCode-8   	  500000	      3884 ns/op
BenchmarkFormat-8     	  300000	      4696 ns/op
PASS
ok  	_/phone-number	5.197s
~~~

Notice the magnitude change in ns/op! That's something to keep an eye out for.

Thanks for reading!
Cheers,
Gergely.
