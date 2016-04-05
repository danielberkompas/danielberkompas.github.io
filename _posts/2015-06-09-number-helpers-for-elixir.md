---
layout: "post"
author: "Daniel Berkompas"
title: "Number Helpers For Elixir"
categories:
  - elixir
date: 2015-06-09 07:00:00 -0800
---

Since I started working on a [Phoenix](http://phoenixframework.org) app, I was frustrated by the lack of number conversion helpers in Elixir/Erlang. I didn't want to have to rewrite `number_to_currency` every time I want to use it.

So, I created [Number](https://github.com/danielberkompas/number). It's basically a shallow clone of NumberHelper from ActionView in Rails.  Now, Elixir users can have `number_to_currency` too!

<!-- more -->

```elixir
import Number.Currency

number_to_currency(nil)
nil

number_to_currency(1000)
"$1,000"

number_to_currency(1000, unit: "£")
"£1,000"

number_to_currency(-1000)
"-$1,000"

number_to_currency(-234234.23)
"-$234,234.23"

number_to_currency(1234567890.50)
"$1,234,567,890.50"

number_to_currency(1234567890.506)
"$1,234,567,890.51"

number_to_currency(1234567890.506, precision: 3)
"$1,234,567,890.506"

number_to_currency(-1234567890.50, negative_format: "(%u%n)")
"($1,234,567,890.50)"

number_to_currency(1234567890.50, unit: "R$", separator: ",", delimiter: "")
"R$1234567890,50"

number_to_currency(1234567890.50, unit: "R$", separator: ",", delimiter: "", format: "%n %u")
"1234567890,50 R$"
```

As my free time permits, I expect to add more features to it, implementing those parts of Rails's NumberHelper that I consider useful to a broader audience.
