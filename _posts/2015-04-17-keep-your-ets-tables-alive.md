---
layout: post
title: "On Keeping Your ETS Tables Alive"
categories:
  - elixir
---

In my ongoing quest to make Elixir libraries that integrate with 
[Twilio][twilio], I found that I needed a lookup table to store the state of 
ongoing calls in.

In Rails, this table would probably be a Postgres table or a list key in Redis. 
But before jumping to one of these familiar solutions, I thought, "What does 
Elixir/Erlang already have that would meet this need?"

<!-- more -->

## Erlang Term Storage (ETS)

I was not disappointed. It turns out that Erlang has an in-memory store called
[ETS][ets], which is perfect for my use case, and here's why.

In Ruby/Rails apps, most of your state is thrown away after every request unless 
you save it in a database. Then, on the next request, you have to load it all up 
again and convert it back into Ruby data types, making the request take longer 
than it needs to. This is because Rails doesn't really offer a persistent 
in-memory store that survives between requests.

I didn't want to have to calculate the previous state of a call or load it out
of a database. After all, I just calculated it! It should still be available
in memory when the user takes another action. The [ETS][ets] store is great for
this because:

1. It persists between requests, and 
2. The data stored in it (regular Erlang tuples) doesn't have to be converted 
   back into Erlang types, so there's very little overhead.

## The Problem: Process Death

There was just one little problem. ETS tables are erased as soon as the process
which owns them dies. So, if there is ever a crash in my table owner, my
entire ETS table will be erased, leaving on-going calls in an inconsistent 
state.

This is a case where Erlang's "Let it crash" philosophy shouldn't be followed.
The OTP supervisor _can_ reboot my crashed process, but the ETS table it owned 
will be gone, making for a poor crash recovery. This is a case where the
defaults need to be improved.

## The Solution: A Table Manager

To prevent this, I set up a very simple `TableManager` process. It has to be
very simple to ensure that _it_ never crashes. This process creates the table,
sets itself as the "[heir][heir]", and then [gives away][give-away] the table 
to a designated consumer process.

If that consumer process ever dies, the `TableManager` will receive back
control, wait until the consumer is rebooted, and then hand it back. The ETS
table stays alive, and everybody's happy.

Since I've been open sourcing my Elixir work so far, I decided to open source
this `TableManager` process. You can find it in my [Immortal][immortal] library
on Github.

## Final Thoughts

I don't need Redis any more!

Seriously, I was impressed at how easy it was to create a durable state storage
system without any external dependencies in Elixir/Erlang. This concept of
persistent state makes my applications feel more like OS programming rather than
transactional, web programming. So far, I like the difference.

[ets]: http://www.erlang.org/doc/man/ets.html
[heir]: http://www.erlang.org/doc/man/ets.html#heir
[give-away]: http://www.erlang.org/doc/man/ets.html#give_away-3
[immortal]: https://github.com/danielberkompas/immortal
[twilio]: http://twilio.com
