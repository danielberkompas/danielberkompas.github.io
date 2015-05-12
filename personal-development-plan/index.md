---
layout: "post"
title: "Personal Development Plan"
author: "Daniel Berkompas"
---
# 2015
<hr />

## Vision
While I am interested in other types of programming, I plan to focus on my 
current forte in 2015: web programming. My personal development plan in 2015 
assumes the following about the future of web technology:

- Customers will continue to demand more interactivity. Javascript frameworks 
  will therefore become even more prevalent.

- Scalability will become a dominant factor in web framework choice. This is
  because successful businesses will have to serve ever increasing volumes of
  traffic.

- New platforms will use the web as a backbone. For example, the "Internet of
  Things" will continue to grow. These new clients will have a strong emphasis
  on real-time data.

- 100% uptime will become extremely important.

- Maintainability will become a focus for businesses investing in technology.
  Recycling their investments every two or three years will become unappealing
  as silly investments fail. The tech bubble will pop.

The plan has three parts. "Concepts" are programming ideas I want to get a more
solid grasp of, "Tools" are libraries and languages I'm interested in learning,
and "Projects" are ideas I plan to implement during the year.

## Concepts

**Algorithms**  
Most web development is just putting a pretty face on a SQL database.
However, with a deep understanding of algorithms, much harder problems can be 
solved, and a different kind of innovation becomes possible. Being self-taught,
I haven't given as much attention to this before as I ought to.

[![Algorithms Manual][algorithms-manual-img]][algorithms-manual]

**Regular Expressions**  
Regular expressions are extremely powerful. Many times, it is possible to
implement a feature with a language-agnostic regular expression rather than
using some specific feature of a specific language. As a result, Regex knowledge
is both _useful_ and _very portable_. In an age where programming languages are
a dime a dozen, I'd really like to have as much permanent knowledge as possible.

[![Mastering Regular Expressions][mastering-regex-img]][mastering-regex]

## Tools

- [Elixir][elixir]: A relatively new functional programming language written on
  top of the Erlang VM, with a Ruby-esque syntax and strong concurrency,
  stability, tooling, and metaprogramming.

  <div class="float-left">[![Metaprogramming Elixir][m-elixir-img]][m-elixir]</div>
  <div class="float-left">[![Programming Elixir][p-elixir-img]][p-elixir]</div>
  <div class="clear"></div>

- [Phoenix][phoenix]: An exciting Elixir web framework with excellent
  performance and real-time features.

- [Ember.js][emberjs] and [Ember CLI][embercli]. With Ember 2.0, I'm pretty
  confident that Ember.js will be a good Javascript framework to spend time and
  energy learning. The team is in this for the long term.

## Projects

- **"OnSale"**. A [Phoenix][phoenix] app capable of monitoring prices
  of products on any website and sending notifications when prices change.
  Contains a significant Javascript component as well.  
  <span class="green">[in-progress]</span>

- **[LeadSimple][leadsimple]**. Render customer UI with Ember.js as a single, 
  unified Ember CLI app, rather than the current hybrid approach.  
  <span class="orange">[unstarted]</span>

## Completed Projects

- **[ExTwiml][extwiml]**. An Elixir library for rendering TwiML for Twilio.
- **[ExTwilio][extwilio]**. An Elixir library for interacting with Twilio's API.
- **[Telephonist][telephonist]**. An Elixir library for handling Twilio phone calls
  using state machines. Depends on [ExTwiml][extwiml] and complements
  [ExTwilio][extwilio].
- **[Phoenix Routes][phoenix-routes]**. A toy [Phoenix][phoenix] app to test
  Phoenix routing code and see the resulting routes immediately.

[leadsimple]: http://leadsimple.com
[phoenix-routes]: http://phoenixroutes.herokuapp.com/
[extwiml]: https://github.com/danielberkompas/ex_twiml
[extwilio]: https://github.com/danielberkompas/ex_twilio
[telephonist]: https://github.com/danielberkompas/telephonist
[algorithms-manual-img]: /assets/img/algorithms-manual.jpg
[algorithms-manual]: http://www.amazon.com/gp/product/1848000693/
[mastering-regex]: http://www.amazon.com/Mastering-Regular-Expressions-Jeffrey-Friedl/dp/0596528124
[mastering-regex-img]: /assets/img/mastering-regex.jpg
[p-elixir]: https://pragprog.com/book/elixir/programming-elixir
[p-elixir-img]: https://imagery.pragprog.com/products/361/elixir_xlargecover.jpg?1368724397
[m-elixir]: https://pragprog.com/book/cmelixir/metaprogramming-elixir
[m-elixir-img]: https://imagery.pragprog.com/products/430/cmelixir_xlargecover.jpg?1415371472
[embercli]: http://ember-cli.org
[emberjs]: http://emberjs.com
[elixir]: http://elixir-lang.org
[phoenix]: http://phoenixframework.org
