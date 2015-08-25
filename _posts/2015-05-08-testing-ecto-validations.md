---
layout: "post"
title: "Testing Ecto Validations"
categories:
  - elixir
date: 2015-05-08 07:00:00 -0800
---

I recently was playing around with [Phoenix][phoenix] and [Ecto][ecto], Elixir's
database library, and I wanted to test my validations. In the process, I wrote a
[little library][Ecto.ValidationCase] along the lines of [Shoulda][shoulda] from 
Ruby.  However, when José Valim saw it, he 
[suggested a much better approach][better] which I think illustrates what makes 
Elixir great.

<!-- more -->

## Validating Ecto Models

In Ecto, a model looks like this:

```elixir
defmodule MyApp.User do
  use Ecto.Model

  schema "users" do
    field :username, :string
    field :email, :string
    field :encrypted_password, :string
  end
end
```

If you want to validate that a new user has a username and email address before
inserting it into the database, you can define a `changeset/2` function:

```elixir
@required_params ~w(username, email)
@optional_params ~w()

def changeset(model, params) do
  model
  |> cast(params, @required_params, @optional_params)
end
```

`cast/4` will return a `Changeset` struct which has `valid?` attribute, which 
can be true or false. It can perform basic presence validation, using a list of
required and optional fields. Everything not in those fields will be ignored.

If you want to perform more advanced validation, Ecto has helpers for that as
well, such as `validate_unique`:

```elixir
def changeset(model, params) do
  model
  |> cast(params, @required_params, @optional_params)
  |> validate_unique(:email)
  |> validate_format(:email, ~r/@/)
end
```

Since all the validators take a changeset as their first argument, they can be
made into pipelines quite easily, like the above.

With a `changeset/2` function in place, you can then design your controller 
action like so:

```elixir
def create(conn, params) do
  user = MyApp.User.changeset(%MyApp.User{}, params)

  if user.valid?
    MyApp.Repo.insert(user)
    # return success
  else
    # render error
  end
end
```

There are several reasons this is great.

1. There are no global validations gumming things up in places they shouldn't.
   If you do Rails long enough, you'll have a time when its global validations
   cause you pain.
2. Testing models is easy, because you don't _have_ to insert valid models in
   every test. Changesets are an opt-in behavior.
3. If you need different validations depending on context, just define multiple
   `changesets`, like this:

```elixir
# :create is an action name, or whatever else you want
def changeset(:create, model, params) do
  model
  |> cast(params, ["username"], [])
end

def changeset(:update, model, params) do
  model
  |> cast(params, ["email"], [])
end
```

## Testing Validations
Testing an Ecto validation is a simple process:

```elixir
# Generate a changeset for a given field/value pair
changeset = MyApp.User.changeset(%MyApp.User{}, %{username: nil})

# Assert that an error is present
assert {:username, "can't be blank"} in changeset.errors

# Or, assert that the validation passed
refute {:username, "can't be blank"} in changeset.errors
```

As José suggested, this can be cleaned up more by adding a simple private 
function to your test case module:

```elixir
defp errors_on(model \\ %MyApp.User{}, params) do
  model.__struct__.changeset(model, params).errors
end
```

You can then write the above like this:

```elixir
assert {:username, "can't be blank"} in errors_on(%{username: nil})
refute {:email, "can't be blank"} in errors_on(%{email: nil})
```

The code is explicit, and there is no magic. What's more, there's no need for a
gem like `shoulda` to clean things up here, because it's simple enough! 

This `errors_on/2` function can easily be extracted to a module and reused, 
just like any other Elixir code. New developers looking at your code will easily
be able to figure out what is going on, unlike `shoulda`, where they must accept
what it's doing on faith.

## A Simpler Way to Code

I love how Elixir encourages this simpler way of coding. There are more moving
parts than there are in Ruby land, perhaps, but they're conceptually simpler.
Everything is made up of state and functions when you get down to it.  It's 
certainly possible to do complex things, but more often than not, you _don't 
have to._

It all comes down to the difference between _explicitness_ and _implicitness_.
Elixir code is meant to be mostly _explicit_, whereas Ruby code often is
_implicit_. Explicit code doesn't hide what it's doing. Implicit code hides its 
implementation and thus has a steeper learning curve should you need to dig into
its innards.

If you write code explicitly, it may be a little longer than the implicit
version, but it will be easier to change, making for a more maintainable
system.

I'm still a recovering Ruby developer, (where we DSL all the things!) but
learning this explicit style has been an enlightening experience so far, and one
I'd recommend to any Rubyist.

## Resources

- [Ecto.Changeset Documentation][Ecto.Changeset]
- [José Valim Taking Me to the Woodshed][better] (It was good, though)

[shoulda]: https://github.com/thoughtbot/shoulda
[better]: https://groups.google.com/forum/#!topic/elixir-lang-talk/kwLLyCiarls
[Ecto.ValidationCase]: https://github.com/danielberkompas/ecto_validation_case
[Ecto.Changeset]: http://hexdocs.pm/ecto/0.11.2/Ecto.Changeset.html
[phoenix]: http://phoenixframework.org
[ecto]: https://github.com/elixir-lang/ecto
