---
layout: post
title: Stream Paginated APIs in Elixir
categories: 
- elixir
- twilio
---

This past week, as I worked on my new [ExTwilio][ExTwilio] API library for
[Twilio][twilio], I ran into a snag dealing with [Twilio's API pagination][twilio-page-api].

Twilio paginates its "list" APIs, requiring multiple requests to fetch all
of a given resource. However, users of my API library will expect to be able to
fetch _all_ of a resource and perform operations on it, like this:

```elixir
calls = ExTwilio.Call.all
Enum.each calls, fn(call) ->
  # perform some operation
end
```

Users won't want to mess with the details of pagination. They want to get a
collection containing everything and then operate on it.

I find that there are two basic ways to achieve this, a _blocking_ way and a
_non-blocking_ way.

## The Blocking Approach
In this first approach, we fetch all the pages, merge their results together, 
and _only then_ return a collection to the caller. The caller is blocked and
must wait while this is going on.

This isn't ideal for two reasons:

1. It will take an unknown amount of time. 
2. Nothing be done with the data until it has _all_ been loaded.

The user probably doesn't want to wait to perform any operations until the
entire result set has been fetched.

## The Non-Blocking Approach
The second, better approach, is to use Elixir's [Stream][Elixir.Stream] module to
create a lazy collection. As we iterate through the collection, the [Stream][Elixir.Stream]
module will automatically fetch new pages of the API as needed.

You can think of the Stream here like a bucket used to fetch water from a well.
The well is the API, and the bucket can hold one page of results at a time. The
Stream returns items from the bucket, one at a time. 

The logic to follow looks like this:

- First, we fill the bucket with the first page of results from the well.
- We then pour out the bucket, one result at a time.
- If the bucket is empty and the well is also empty, stop.
- If the bucket is empty and the well still has water, fill the bucket again and
  resume pouring.

Results from the well can be processed as each "bucket" arrives, and the bucket
will be refilled as long as there are still results left.

How do we implement this in Elixir? We start by defining a `stream` function.

```elixir
def stream
  start = fn -> end
  next_item = fn -> end
  stop = fn -> end

  Stream.resource(start, next_item, stop)
end
```
The [Stream.resource/3][Stream.resource] method takes three funs. 

- `start` is used to initialize the Stream, and set its initial state.
- `next_item` takes the Stream's previous state and returns a tuple, like so:
  {[next item for collection], new_state}
- `stop` is a handy hook to clean up once the Stream is finished.

We'll look at each of these in order.

## start

```elixir
start = fn ->
  # Assumes there is a list/0 method that will fetch the first page
  # of the API, in the format {status, items, paging_metadata}
  case list do
    {:ok, items, meta} -> {items, meta["next_page_uri"]}
    _error             -> {[], nil}
  end
end
```

We will store our state for the Stream in a two-element tuple, in this format:
`{items, next_page_url}`. If our `list/0` method returns the first page
successfully, we'll fill the state with that first page of items.

If, on the other hand, the first page doesn't load, we'll start with an empty
state which we'll make cause the Stream to finish instantly.

## next_item

This is where things get interesting.

```elixir
next_item = fn state = {[], nil, _}     -> {:halt, state}
               state = {[], next, opts} -> fetch_next_page.(state)
               state                    -> pop_item.(state)
end
```

The logic is pretty simple:

- If there are no items left in the current page, and there is no next page, then stop.
- If there are no items left in the current page, but there is a next page, then go get
  that page.
- If neither of the above are true, pop an item off the state and return it.

You'll notice there are two new funs here, `fetch_next_page` and `pop_item`.
They look like this:

```elixir
# Use pattern matching to pop the top item off the list of items, passing the 
# tail as the new state.
pop_item = fn {[head|tail], next} ->
  new_state = {tail, next}
  {[head], new_state}
end

# Get the next page, and use pop_item to both set the new state and return the
# first item of the new page.
fetch_next_page = fn state = {[], next_page} ->
  # Assumes you have a method called `fetch_page` which will take a page URL and
  # get that page of results.
  case fetch_page(next_page) do
    {:ok, items, meta} -> pop_item.({items, meta["next_page_uri"]})
    {:error, _msg}     -> {:halt, state}
  end
end
```

## stop
The simplest function of all! We don't really have anything to clean up.

```elixir
stop = fn(_state) -> end
```

## Putting It All Together

Our complete `stream/0` method looks like this:

```elixir
def stream
  start = fn ->
    # Assumes there is a list/0 method that will fetch the first page
    # of the API, in the format {status, items, paging_metadata}
    case list do
      {:ok, items, meta} -> {items, meta["next_page_uri"]}
      _error             -> {[], nil}
    end
  end

  # Use pattern matching to pop the top item off the list of items, passing the 
  # tail as the new state.
  pop_item = fn {[head|tail], next} ->
    new_state = {tail, next}
    {[head], new_state}
  end

  # Get the next page, and use pop_item to both set the new state and return the
  # first item of the new page.
  fetch_next_page = fn state = {[], next_page} ->
    # Assumes you have a method called `fetch_page` which will take a page URL and
    # get that page of results.
    case fetch_page(next_page) do
      {:ok, items, meta} -> pop_item.({items, meta["next_page_uri"]})
      {:error, _msg}     -> {:halt, state}
    end
  end

  next_item = fn state = {[], nil, _}     -> {:halt, state}
                 state = {[], next, opts} -> fetch_next_page.(state)
                 state                    -> pop_item.(state)
  end

  stop = fn(_state) -> end

  Stream.resource(start, next_item, stop)
end
```

You can now stream through resources to your heart's content! For example, you
can process all of your Twilio Calls like this:

```elixir
Enum.each ExTwilio.Call.stream, fn(call) ->
  puts call.sid
end
```

Since it's just a Stream, you can also _stack operations_ on the stream which
will be lazily executed whenever you want!

```elixir
# Find calls that were longer than 2 minutes (160 seconds)
# "stream" is returned instantly
stream = Stream.filter ExTwilio.Call.stream, fn(call) -> 
  call.duration > 120 
end

# Append another operation to get just the call SIDs. Only calls that got
# through the filter above will be mapped.
stream = Stream.map stream, fn(call) -> call.sid end

# Execute all the operations and return a collection:
Enum.into stream, [] # => ["CN234098123....", "CN123098123..."]

# Or, just put out all the sids
Enum.each stream, fn(sid) ->
  IO.puts sid
end
```

Using this technique, you can programmatically build up filters on your remote
paged resources, knowing that all those filters will only be executed when you
need the final data. (Aside: this is very similar to ActiveRecord::Relation)

In short, this is very cool. But you knew that.

## Wrapping Up

[ExTwilio][ExTwilio] supports streaming for all Twilio endpoints that have a
paged API. For those masochistic users who still want a blocking method, I've
given them an `all` method as well. Internally, it just delegates to `stream`:

```elixir
def all
  Enum.into stream, []
end
```

Writing this code was a lot of fun, and I hope this walkthrough was helpful to 
you! If you want to learn more about Elixir's Stream API, check out the 
following:

- [Stream Docs][Elixir.Stream]
- [Programming Elixir, by Dave Thomas][programming-elixir]

If you want to keep up with my progress building full-featured Elixir libraries
for Twilio, stay tuned to this blog, or check out [ExTwilio][ExTwilio] and
[ExTwiml][ExTwiml].

[ExTwilio]: https://github.com/danielberkompas/ex_twilio
[ExTwiml]: https://github.com/danielberkompas/ex_twiml
[twilio]: https://www.twilio.com
[twilio-page-api]: https://www.twilio.com/docs/api/rest/response#response-formats-list
[Elixir.Stream]: http://elixir-lang.org/docs/stable/elixir/Stream.html
[Stream.resource]: http://elixir-lang.org/docs/stable/elixir/Stream.html#resource/3
[programming-elixir]: https://pragprog.com/book/elixir/programming-elixir
