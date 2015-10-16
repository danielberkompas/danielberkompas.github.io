---
layout: "post"
title: "Why I'm Checking Out GraphQL"
author: "Daniel Berkompas"
---

Like many other Elixir programmers, I had heard rumblings about 
[GraphQL][graphql-spec], but only recently decided to take a deeper look at it 
after Chris McCord mentioned it in his [recent talk at ElixirConf 2015][phoenix-next].

I think it shows great promise and I want to help out with implementing an
Elixir version, and I'll explain why. But first, let me outline some of the
problems with the way we build REST APIs today.

## The Mess We're In

It's increasingly common to serve a Javascript front-end app from a REST API
these days. The merits of Javascript apps vs. server-rendered apps have been
debated ad nauseum and I'm not going to repeat that debate here.

Suffice to say, I believe there _is_ something qualitatively different _and
better_ about the kind of user interfaces that client-side Javascript can
create, which servers cannot match. This means that I believe it is desirable in
many cases to have a client-rendered interface, which then raises the question
of how we ship data between the client and the server.

REST has been a decent solution, but when compared to GraphQL, it suffers from 
a number of severe problems.

### REST is a Large, Complicated Middleman

An API should serve three purposes: 

1. Normalize your data into a predictable schema
2. Authorization
3. Apply business logic to user input

Everything beyond this is just bloat, and REST API servers tend to get clogged 
up with a lot of other concerns.

I think we should ask ourselves this question more often: "Do we even need a
server here? Could we implement this app by just talking straight to Postgres?"
Most of the time the answer would be no, but the mental exercise would help keep
the API server you do write as tiny and simple as possible. It really is a
database wrapper, after all.

### REST APIs are Rigid, Not Responsive

Implementing a REST API is a complicated and rigid process, because you need 
to specify all the URL endpoints, as well as the format of the response for each
resource and endpoint, before you build anything.

While you do this designing, you make assumptions about what data the
client interface will need to fetch, and how. Many of these assumptions will
turn out to be wrong, and you'll have to modify your design.

Since REST APIs are rigid _by design_, changing your API can mean creating a 
whole new version of your API just to support one feature you didn't think of in
the beginning. This gets expensive fast.

This is also exactly the opposite of the way it should be. The server exists to 
serve the client, so it needs to be as responsive as it possibly can be to the 
needs of the client. Instead, today, we tend to implement our clients around the
server, because the server is the more rigid part of the system and would be
the most difficult to change.

### REST APIs are Wasteful

Network bandwidth is still a limited commodity, and minimizing the amount of 
data that travels over the wire makes your app render faster and the experience
smoother. However, REST APIs dictate what data can and must be sent for each
request, regardless of what the client actually needs in order to render
something to the user. 

As a result, APIs serve a bunch of data that isn't needed, thereby wasting all
the time it takes to fetch and transmit that data. Also, due to the way REST is
designed, you often have to make multiple requests to get the data you need,
which is also wasteful.

We need a better way to batch requests and limit the data returned.

### Documenting a REST API is Hard

If you write a REST API, you really should document it, even if it's only for
internal use! If someone other than you will ever use it, you need to document
it to save them the effort of having to read through your potentially
complicated source code. Especially if _they don't even have access to your
source code._

The trouble is, documentation is hard to do right. [Stripe][stripe-docs] hit it
out of the park with the quality of their docs, but few others have ever done
that good of a job. That's because it's hard.

## How GraphQL Helps

In GraphQL, the client is in charge. The client specifies what data it needs,
and the server does only what is necessary to return that data. 

The server becomes a lightweight app dedicated to resolving GraphQL queries into
data, with a minimum of other concerns. It also becomes much more agile, because
it's much easier to expose more data on the server without breaking or slowing
down existing requests. This allows you to elegantly expand your API as your
knowledge of your clients' needs grows.

Both queries and mutations can be batched, allowing for maximum network
efficiency. If you need even more efficiency, I don't see any reason you
couldn't use something like [MessagePack][message-pack] as your data format
rather than JSON.

Further, the server is self-documenting, because it broadcasts its schema to the
outside world. This also allows the client to make sure that its requests are 
valid before it even tries to make them.

If all that intrigues you, GraphQL is easy to learn. I've found
the [Learn GraphQL website][learngraphql] to be very helpful. Elixir support
isn't quite there yet, but I hope to help out with that.

[learngraphql]: http://learngraphql.com
[message-pack]: http://msgpack.org/
[stripe-docs]: https://stripe.com/docs/api
[graphql-spec]: http://facebook.github.io/graphql/
[phoenix-next]: http://confreaks.tv/videos/elixirconf2015-what-s-next-for-phoenix
