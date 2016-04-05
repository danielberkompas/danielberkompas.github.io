---
layout: "post"
title: "Rate Limiting a Phoenix API"
author: "Daniel Berkompas"
categories:
  - elixir
date: 2015-06-16 07:00:00 -0800
---

In my spare time, I've been working on a little [Phoenix][phoenix] project that involves a JSON API. Developers frequently neglect rate limiting when they build an API, assuming they are even aware that it is a best practice.

It's true that in many cases rate limiting isn't worth the effort, but when it comes to authentication, it definitely is. For example, the recent high-profile [iCloud][icloud] security breach which released celebrity photos in to the internet could have been prevented had Apple implemented rate limiting on one of their authentication APIs. This would have prevented the brute-force attack that the hackers used to guess the celebrities' passwords.

<!-- more -->

My little API isn't likely to hold any sensitive information, but I decided to rate limit my authentication API anyway. Here's how I did it.

## Introducing Plugs

In [Phoenix][phoenix], the request lifecycle is handled through "plugs", which are simple functions or modules that take a connection, modify it, and then return the modified connection. Plugs come from an Elixir core project, aptly named "[Plug][plug]".

If you're familiar with Ruby, plugs work very similarly to Rack middleware. However, instead of being a rare form of voodoo like Rack sometimes is to Ruby developers, plugs are very accessible in Elixir/Phoenix, and are used often.

In its simplest form, a plug looks like this:

```elixir
def my_custom_plug(conn, options \\ []) do
  conn
end
```

This plug does nothing. It just takes a connection and returns it, which is all a good little plug _has_ to do. To use this plug in a Phoenix controller, you just have to import the function and add it to the controller's plug pipeline:

```elixir
defmodule MyApp.Controller do
  use MyApp.Web, :controller
  import MyPlug, only: [my_custom_plug: 2]

  plug :my_custom_plug
  plug :action
end
```

The `my_custom_plug` function will then run on the connection before the
controller action is called.

## The RateLimit Plug

The [Plug][plug] interface is ideal for rate limiting because it allows us to intercept the request before any real work is done. This prevents rejected requests from producing any serious load on the server.

For my plug, I used a library called [ExRated][ex_rated] to do the brunt of the rate limiting work. It has a simple API:

```elixir
ExRated.check_rate("bucket_name", interval_in_milliseconds, maximum_requests)
# => {:ok, count} or {:fail, count}
```

Using this, it's very easy to construct a plug:

```elixir
defmodule MyApp.RateLimit do
  import Phoenix.Controller, only: [json: 2]
  import Plug.Conn, only: [put_status: 2]

  def rate_limit(conn, options \\ []) do
    case check_rate(conn, options) do
      {:ok, _count}   -> conn # Do nothing, allow execution to continue
      {:fail, _count} -> render_error(conn)
    end
  end

  defp check_rate(conn, options) do
    interval_milliseconds = options[:interval_seconds] * 1000
    max_requests = options[:max_requests]
    ExRated.check_rate(bucket_name(conn), interval_milliseconds, max_requests)
  end

  # Bucket name should be a combination of ip address and request path, like so:
  #
  # "127.0.0.1:/api/v1/authorizations"
  defp bucket_name(conn) do
    path = Enum.join(conn.path_info, "/")
    ip   = conn.remote_ip |> Tuple.to_list |> Enum.join(".")
    "#{ip}:#{path}"
  end

  defp render_error(conn) do
    conn
    |> put_status(:forbidden)
    |> json(%{error: "Rate limit exceeded."})
    |> halt # Stop execution of further plugs, return response now
  end
end
```

You can then use this plug to rate limit specific actions in your controller like so:

```elixir
import MyApp.RateLimit

plug :rate_limit, max_requests: 5, interval_seconds: 60 when action in [:create]
plug :action
```

## Customizing for Authentication

The RateLimit plug as it currently exists is great for most use cases, but it still isn't adequate for authentication. This is because limiting based on IP address still allows an attacker to make thousands of attempts on a user account from different computers.

To protect against this, we need to rate limit login attempts based on the _username_, not the IP address. The plug needs an optional `:bucket_name` parameter, so that this can be customized.

```elixir
defp check_rate(conn, options) do
  interval_milliseconds = options[:interval_seconds] * 1000
  max_requests = options[:max_requests]
  bucket_name = options[:bucket_name] || bucket_name(conn)

  ExRated.check_rate(bucket_name, interval_milliseconds, max_requests)
end
```

We then add a new function to our controller, setting the dynamic `:bucket_name` to "authorization:{email}":

```elixir
def rate_limit_authentication(conn, options \\ []) do
  options = Dict.merge(options, [bucket_name: "authorization:" <> conn.params.email])
  MyApp.RateLimit.rate_limit(conn, options)
end
```

And use this instead of the old `:rate_limit` plug:

```elixir
import MyApp.RateLimit

plug :rate_limit_authentication, max_requests: 5, interval_seconds: 60
plug :action
```

This will then rate limit requests against the user's email address, not the IP address that the requests are coming from. So, no matter how many IPs the hacker has access to, he can't get around the rate limit.

## The Finished Plug

```elixir
defmodule MyApp.RateLimit do
  import Phoenix.Controller, only: [json: 2]
  import Plug.Conn, only: [put_status: 2]

  def rate_limit(conn, options \\ []) do
    case check_rate(conn, options) do
      {:ok, _count}   -> conn # Do nothing, pass on to the next plug
      {:fail, _count} -> render_error(conn)
    end
  end

  defp check_rate(conn, options) do
    interval_milliseconds = options[:interval_seconds] * 1000
    max_requests = options[:max_requests]
    bucket_name = options[:bucket_name] || bucket_name(conn)

    ExRated.check_rate(bucket_name, interval_milliseconds, max_requests)
  end

  # Bucket name should be a combination of ip address and request path, like so:
  #
  # "127.0.0.1:/api/v1/authorizations"
  defp bucket_name(conn) do
    path = Enum.join(conn.path_info, "/")
    ip   = conn.remote_ip |> Tuple.to_list |> Enum.join(".")
    "#{ip}:#{path}"
  end

  defp render_error(conn) do
    conn
    |> put_status(:forbidden)
    |> json(%{error: "Rate limit exceeded."})
    |> halt # Stop execution of further plugs, return response now
  end
end
```

## Performance

The performance of this plug is really stunning. On my relatively modest Macbook Pro, I'm seeing sub-millisecond response times when the rate limit kicks into effect. Rails devs, let that sink in for a second.

_Sub-millisecond response times..._

So, using this plug doesn't slow down valid requests in any measurable way, and with response times like that, it will be hard to overwhelm your API with bogus traffic.

## Conclusion

I was pleasantly surprised at how easy this was to implement and how elegant the solution ended up being. Let's review what we implemented:

- A rate limiter that can be customized per action and per controller.
- Without any external dependencies. (I'm looking at you, Redis)
- With sub-millisecond response times.

Elixir and Phoenix continue to impress me! This exercise also made me think that I should look more into Rack in Ruby-land. It's probably under-utilized for problems like this. Perhaps that will be the subject of a future post. For now though, so long!

[ex_rated]: http://hex.pm/packages/ex_rated
[icloud]: http://icloud.com
[plug]: https://github.com/elixir-lang/plug
[phoenix]: http://phoenixframework.org
