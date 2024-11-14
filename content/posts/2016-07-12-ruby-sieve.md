+++
author = "hannibal"
categories = ["Ruby"]
date = "2016-07-12"
title = "Ruby Sieve"
url = "/2016/07/12/ruby-sieve"

+++

Though it could be done better, I'm sure, but I'm actually pretty satisfied with this one. It loops only twice as opposed to filtered ranges and whatnot other solutions to the sieve. I was thinking of rather creating a list and deleting elements from it, but that's already three loops.

Maybe I'll do a benchmark later on more solutions.

~~~ruby
# Sieve contains a function to return a set of primes
class Sieve
  def initialize(n)
    @n = n
  end

  # Returns a list of primes up to a certain limit
  # @param n limit
  # @return list of primes
  def primes
    marked = []
    primes = []
    (2..@n).each do |e|
      unless marked.include?(e)
        primes.push e
        (e..@n).step(e) { |s| marked.push s }
      end
    end
    primes
  end
end
~~~

Cheers,
Gergely.
