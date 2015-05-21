---
layout: "post"
author: "Daniel Berkompas"
title: "Useful Ecto Validators"
categories:
  - elixir
---

Over the past week, I've created a couple custom validators for my Elixir
projects which use [Ecto][ecto]. Since validators are just functions that take a
changeset and return a changeset, they're very easy to write.

I'll update this post in the future if I find or write any more useful
validators, or refactor ones I've already written.

## validate_url

This validator will ensure that a URL is parseable by the Erlang `http_uri`
module. I obviously could have gone for a huge regular expression here, but this
seemed a little more trustworthy.

```elixir
def validate_url(changeset, field, options \\ []) do
  validate_change changeset, field, fn _, url ->
    case url |> String.to_char_list |> :http_uri.parse do 
      {:ok, _} -> []
      {:error, msg} -> [{field, options[:message] || "invalid url: #{inspect msg}"}]
    end
  end
end
```

### Usage

```elixir
validate_url(changeset, :url, message: "URL is not a valid URL!")
```

## validate_array_inclusion

I wrote this validator because of PostgreSQL's Array field type. Ecto already has
a `validate_inclusion` validator which will ensure that a given value is a
member of a given array. However, in order to validate a Postgres array field, I
needed to validate that all _members_ of array A are also members of array B.

This function does the trick:

```elixir
def validate_array_inclusion(changeset, field, allowed, options \\ []) do
  validate_change changeset, field, fn _field, values ->
    valid? = Enum.all? for value <- values, do: value in allowed
    if valid?, do: [], else: [{field, options[:message] || "is invalid"}]
  end
end
```

### Usage

```elixir
validate_array_inclusion(changeset, :notify, ~w(email sms phone), message: "custom message here")
```

This will ensure that the "notify" array only contains some subset of the values
`["email", "sms", "phone"]`.


[ecto]: https://github.com/elixir-lang/ecto
