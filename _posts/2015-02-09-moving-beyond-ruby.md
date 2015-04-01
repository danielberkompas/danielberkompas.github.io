---
layout: post
title: Moving Beyond Ruby
categories: ruby elixir
---

I've been a Ruby developer for the last 4 years of my career. It's served me
very well, I still like it, and I expect that I'll still be writing it for my
career for years to come. However, there are some things that Ruby (and Rails)
don't do so well out of the box, and these things are causing me to look
elsewhere.

<!-- more -->

- **Concurrency**. Ruby wasn't built for concurrency, and therefore there is no
  consensus on how to do it.
- **Scalability**. It is genuinely hard to scale Rails applications out of the
  box. In fact, the box should be labeled "Scalability not included." Much of 
  this has to do with Ruby's concurrency support and slow execution speed.
- **Portability**. Anyone who has set up a machine for Ruby development can
  attest to the difficulty of getting up and running. Setting up your production
  servers to run it can also be a pain. (Unless you get to use [Heroku][heroku])

I recognize that there are solutions to all of these problems. However, I would
prefer to be able to write in a language and framework that does a better job of
these things out of the box.

## Going Functional
As a programming teacher, I've seen first hand how students struggle with the 
concept of "Objects" and "Instances." Many of us were taught that Object-Oriented
Programming is easier to learn, yet these students have a hard time getting their
minds around it.

This got me thinking, is Object-Oriented Programming really _simpler_ than
Functional Programming? My working theory is no, it isn't. Programming 
really has to do with transforming data from one state into another state, and 
this fits very well into the functional paradigm of immutable state, first-class
functions, and function chaining.

In contrast, Objects obscure what's really going on. Since computers 
aren't like people, imposing this anthropomorphic system on top of what the 
computer does actually makes things _more complex_.

Theoretically then, we should be able to compose programs that are both
_simpler_ and _easier to understand_ with just the simple building blocks that
Functional Programming languages give us.

## Elixir
To test this theory, I've started learning [Elixir][elixir]. There are, of 
course, other functional languages I could learn, but Elixir seems like the 
easiest to pick up because it superficially looks like Ruby.

See for yourself:

#### Ruby
```ruby
class PagesController < ApplicationController

  def index
    @title = params[:title]
    @members = [
      {name: "Chris McCord"},
      {name: "Matt Sears"},
      {name: "David Stump"},
      {name: "Ricardo Thompson"}
    ]
    render "index"
  end
end
```

#### Elixir
```elixir
defmodule Benchmarker.Controllers.Pages do
  use Phoenix.Controller

  def index(conn, %{"title" => title}) do
    render conn, "index", title: title, members: [
      %{name: "Chris McCord"},
      %{name: "Matt Sears"},
      %{name: "David Stump"},
      %{name: "Ricardo Thompson"}
    ]
  end
end
````

Elixir runs on the Erlang VM, which is battle tested and has excellent support
for concurrency. Also, it has [support for
unikernels](http://elixir-lang.org/blog/2013/05/02/elixir-on-xen/), something
I'm interested in from a security perspective.

I intend to post more about my experiences with Elixir, so stay tuned.
In the future, I also intend to take a look at Rust and Go.

[elixir]: http://elixir-lang.org
[heroku]: http://heroku.com
