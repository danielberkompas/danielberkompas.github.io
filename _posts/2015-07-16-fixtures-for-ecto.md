---
layout: "post"
author: "Daniel Berkompas"
title: "Fixtures for Ecto"
categories:
  - elixir
---

When you test an [Elixir][elixir] app that uses [Ecto][ecto], you will find
yourself needing a way to insert test data into the database. There are many
different approaches to doing this, and I thought I'd cover a few, and then
describe what I think the best approach is for Elixir.

<!-- more -->

## Global Fixtures

By default, Rails solves this problem by automatically inserting a set of rows
into the database before every test case is executed. These rows are configured
by YAML files in the `test/fixtures` directory.

While you could probably hook into `mix test` to insert your data before every
test, I don't think the Rails approach should be imitated here, for the
following reasons.

**First, global fixtures are _implicit_ behavior.** There's nothing in your test
file that tells you where the data is coming from. It's all happening behind the
scenes, and you have to know to go look in the `fixtures/` directory to figure
out what's going on.

In contrast, Elixir emphasizes _explicit_ behavior. It should be obvious where 
the fixtures are coming from, or at least there should be a breadcrumb trail 
_in my test file_ that I can follow to figure that out.

**Second, global fixtures can be brittle.** It assumes that your app can be
adequately tested with one or only a few versions of data. This is often a bad
assumption.

**Third, global fixtures are unecessary performance overhead.** Inserting the
data that you need for _all_ your tests before _every_ test is unecessary. It
may not slow down your tests by much, but why slow them down at all?

**Fourth, YAML is a bad DSL.** Why write data for your ActiveRecord models in 
YAML? Why not just explicitly call `Model.create`? Rails itself acknowledges 
that YAML is a bad DSL by making the `db/seeds.rb` file a plain Ruby file where
you use `Model.create` to create data. (The dichotomy between the fixtures YAML 
and the seeds.rb file confuses every Rails newbie I've taught!)

How exactly is this:

```yaml
daniel:
  name: "Daniel Berkompas"
  email: "test@example.com"
```

Better than this?

```ruby
User.create name: "Daniel Berkompas", email: "test@example.com"
```

Of course, the reason Rails does this is because inserting a row through
ActiveRecord is _slow_, and since all the data has to be inserted before _every_
test, fixtures would slow down your test suite a lot if it went through
ActiveRecord.

## Factory Girl

Many Rails developers use [FactoryGirl][factory_girl] instead of Rails' built in
fixtures. Using FactoryGirl, you define your fixtures using a DSL in Ruby files
under `factories/model_name.rb`.

```ruby
FactoryGirl.define do
  factory :user do
    name "Daniel Berkompas"
    email "test@example.com"
  end
end
```

Then, to create the data:

```ruby
user = FactoryGirl.create(:user)
```

This is better than Rails' default in several respects. The fixtures are not
loaded globally, and the code is more explicit. A user knows that they need to
look for "factory" related stuff to figure out what values `:user` has.

However, it still has two faults which annoy me in the projects where I use it.
First, since it runs through ActiveRecord, it can be very slow to insert data.
Second, the DSL seems unnecessary. How is the above better than this?

```ruby
User.create name: "Daniel Berkompas", email: "test@example.com"
```

I would prefer to see the code be still more explicit, fast, and without any
DSLs, so that new developers don't have to learn a new DSL to write tests.

## Fixtures for Elixir

I suggest that you add a simple `MyApp.Fixtures` module to your `test/support`
directory in your mix project. It could look something like this:

```elixir
defmodule MyApp.Fixtures do
  alias MyApp.User

  def fixture(:user) do
    %User{name: "Daniel Berkompas", email: "test@example.com"}
  end
end
```

If you wanted, the `fixture/1` function could also automatically insert the
`User` record by calling `Repo.insert!/1`:

```elixir
defmodule MyApp.Fixtures do
  alias MyApp.Repo
  alias MyApp.User

  def fixture(:user) do
    Repo.insert! %User{name: "Daniel Berkompas", email: "test@example.com"}
  end
end
```

It's trivial to add support for other models, and even associations:

```elixir
defmodule MyApp.Fixtures do
  alias MyApp.Repo
  alias MyApp.User
  alias MyApp.Post

  def fixture(:user) do
    Repo.insert! %User{name: "Daniel Berkompas", email: "test@example.com"}
  end

  def fixture(:post, assoc \\ []) do
    user = assoc[:user] || fixture(:user)
    Repo.insert! %Post{
      title: "Fixtures for Ecto"
      user_id: user.id
    }
  end
end
```

You can then use this `Fixtures` module in your test like so:

```elixir
defmodule MyApp.MyTest do
  use ExUnit.Case
  import MyApp.Fixtures

  test "some behavior" do
    user = fixture(:user)
    post = fixture(:post, user: user)

    # test logic here
  end
end
```

## Benefits

I think this approach provides several benefits:

- **Simplicity**. There are no DSLs to learn. This means that new developers will
  be able to get up to speed with it very fast.

- **Explicitness**. There's no mystery here. I can look at the test file and know
  exactly how the data is getting created.

- **Speed**. Each test inserts only the data it needs. `Repo.insert!/1` is also
  much more performant than `ActiveRecord::Base#create`, since it's not worrying
  about validations. Ecto model structs are pretty lightweight compared to 
  ActiveRecord objects, so that helps too.

- **Customization**. Since we're only dealing with modules and functions here,
  you can compose your fixtures however you like. Nothing's off limits because
  the DSL doesn't support it, because there is no DSL!

On the whole, I think this simpler approach is much better than either Rails
fixtures or FactoryGirl, and you don't need to add a dependency to get it!

[ecto]: https://github.com/elixir-lang/ecto
[elixir]: http://elixir-lang.org
[factory_girl]: https://github.com/thoughtbot/factory_girl
