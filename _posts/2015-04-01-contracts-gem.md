---
layout: post
title: "Contracts: Type Checking for Ruby"
categories:
  - elixir
  - ruby
date: 2015-04-01 07:00:00 -0800
---

Like many Rubyists who read popular coding news, I recently came across the [Contracts][contracts] gem. It caught my eye because it implements some of the features I like in [Elixir][elixir], but for Ruby.

<!-- more -->

## Type Annotations in Elixir

Both Ruby and Elixir are duck-typed languages. However, Elixir (and its parent, Erlang) allow for static type analysis through annotations. In Elixir, they look like this:

```elixir
@spec add(integer, integer) :: integer
def add(a, b) do
  a + b
end
```

Custom types can be defined:

```elixir
@type uuid :: String.t

@spec find(uuid) :: map
def find(uuid) do
  # implementation here
end
```

Arguments can be deconstructed via pattern matching in type specs, just like they can in function definitions. Suppose the `add/2` function took a tuple instead of two arguments. You could write it like this:

```elixir
@spec add({integer, integer}) :: integer
def add({a, b}), do: a + b
```

All together, your `Math` module might look like this:

```elixir
defmodule Math do
  @spec add({integer, integer}) :: integer
  def add({a, b}), do: add(a, b)

  @spec add(integer, integer) :: integer
  def add(a, b), do: a + b
end

Math.add(2, 3)   # => 5
Math.add({2, 3}) # => 5
Math.add("hello") # => Error: no matching function definition
```

In Elixir, these type annotations are analyzed at compile-time for some errors, and the compiled BEAM file can also be run through [Dialyzer][dialyzer] for static analysis. Together, this two step process can find type errors _without even running your code,_ very similarly to statically typed languages.

## Type Annotations in Ruby using Contracts

The [Contracts gem][contracts] is able to do a lot of the same kind of type annotation that Elixir can do. It even adds support for multiple definitions of the same method!

```ruby
require "contracts"

class Math
  include Contracts

  # Tuple version
  # Ruby doesn't have tuples, so we'll use a two-element array
  Contract [Num, Num] => Num
  def add(numbers)
    add numbers[0], numbers[1]
  end

  # Two arguments version
  Contract Num, Num => Num
  def add(a, b)
    a + b
  end
end

math = Math.new
math.add(1, 2)      # => 3
math.add([1, 2])    # => 3
math.add([1, 2, 3]) # => Error, no matching contract
math.add("hello")   # => Error, no matching contract
```

It also adds some of the pattern matching goodness that Elixir brings to the table. Suppose I had a function `handle_response`, that is supposed to deal with a response from a web API. In Elixir, that function might look like this:

```elixir
def handle_response(:error, body) do
  # Handle error case
end

def handle_response(:success, body) do
  # Handle success case
end
```

In Ruby without the Contracts gem, it would look like this:

```ruby
def handle_response(status, body)
  case status
  when :error then # handle error
  when :success then # handle success
  end
end
```

With the Contracts gem, it looks a lot more like Elixir:

```ruby
Contract :error, String => Any
def handle_response(status, body)
  # handle error
end

Contract :success, String => Any
def handle_response(status, body)
  # Handle success case
end
```

## Important Differences between Contracts and Elixir

There are a few important differences between the [Contracts][contracts] gem and Elixir's type annotations.

| Feature          | Contracts                      | Elixir                         |
| :--------------- | :----------------------------: | :----------------------------: |
| Compile-time     | <span class="red">N/A</span>   | <span class="green">Yes</span> |
| Static analysis  | <span class="red">No</span>    | <span class="green">Yes</span> |
| Violation errors | <span class="green">Yes</span> | <span class="green">Yes</span> |
| Pattern matching | <span class="green">Yes</span> | <span class="green">Yes</span> |
| Custom types     | <span class="green">Yes</span> | <span class="green">Yes</span> |

Since Ruby isn't compiled, the contracts are not statically analyzed prior to running your code. You won't catch a contract error before it occurs, so your tests would still need to exercise most of your code.

## Benefits

Why should you want type annotations? I can think of several reasons:

- **They're optional**. You don't have to use annotations when you want
  duck-typing. But when types are actually important, you can use them.
- **Type errors do exist.** Even if they don't matter to [@dhh][dhh].
- **Contracts provide free documentation.** Very few functions can handle _any_ input. This means that developers always have to look up what types the function can handle, and if you document that, it will make their lives easier. Documentation can easily be generated off these annotations.
- **Code with greater confidence.** You will know that functions won't run without receiving valid input, cutting down on the number of bad things that could occur.

Overall, I think the [Contracts][contracts] gem looks great, and I'm excited to use it in one of my next Ruby projects.

## Additional Resources

- [Contract Gem Tutorial](http://egonschiele.github.io/contracts.ruby/)
- [Contracts.coffee](http://disnetdev.com/contracts.coffee/)

P.S. This is _not_ an April Fool's joke.

[dhh]: http://twitter.com/dhh
[dialyzer]: http://www.erlang.org/doc/man/dialyzer.html
[elixir]: http://elixir-lang.org
[contracts]: https://rubygems.org/gems/contracts
