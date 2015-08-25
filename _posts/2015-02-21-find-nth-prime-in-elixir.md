---
layout: post
title: Find the Nth Prime in Elixir
categories:
  - elixir
date: 2015-02-21 07:00:00 -0800
---

I've been working through [Exercism's][exercism] set of code challenges for 
Elixir, and came across this one.

> Write a program that can tell you what the nth prime is.
>
> By listing the first six prime numbers: 2, 3, 5, 7, 11, and 13, we can
> see that the 6th prime is 13.

This is a perfect use case for Elixir's [Stream][elixir-stream] module, because
we want to generate a list of values and return the last one. 

<!-- more -->

Stream contains a function called `iterate`, which takes a starting value, and 
then lazily generates new value based on the previous value. To create a Stream
for primes, therefore, we can write something like this:

```elixir
# Starting at 2, create new values using the next_prime/1 function
Stream.iterate(2, &next_prime/1)
```

We can then chain a couple of other functions to generate only up to the number
of primes requested, and return the last one.

```elixir
def nth(count) do
  Stream.iterate(2, &next_prime/1)
    |> Enum.take(count)
    |> List.last
end
```

Finally, we must define the `next_prime/1` function. Given a starting point, it
should iterate until it reaches another prime number.

```elixir
def next_prime(num) do
  num = num + 1

  # If the factors only include this number, it's prime
  if factors_for(num) == [num] do
    num
  else
    next_prime(num)
  end
end
```

`factors_for/1` is also a function I defined, but I'll leave it as an exercise
for the reader.

I enjoyed getting to use [Stream][elixir-stream] for the first time with this
exercise. Ruby has something similar ([Enumerable::Lazy][lazy]), but I hadn't
used it or anything like it before. Another tool for the toolbox!

[lazy]: http://railsware.com/blog/2012/03/13/ruby-2-0-enumerablelazy/
[elixir-stream]: http://elixir-lang.org/docs/stable/elixir/Stream.html
[exercism]: http://exercism.io
