---
layout: "post"
author: "Daniel Berkompas"
title: "Replace Callbacks with Ecto.Multi"
comments: true
---

We all have logic in our applications like this:

- When a user is created, send a notification to an admin
- When a post is deleted, remove it from the search cache
- When a password is reset, log out that user's active sessions

These side effects need to be predictable and reliable. Often, they're some of the key business logic of the whole application.

<!-- more -->

Many Rails applications store this kind of logic in their models, using ActiveRecord callbacks. However, since [Ecto 2.0 removed callbacks][remove-callbacks], this is not a viable (or desirable) approach in Elixir.

Instead, Elixir follows a simple rule:

> **If you have shared business logic, create a shared module.**

One of the ways you can do this is by building simple "Service" modules around [Ecto.Multi][ecto-multi].[^1] Since business logic usually revolves around an [Ecto.Schema][ecto-schema], I often name service modules after the schema they interact with, such as `UserService` or `PostService`.

```elixir
defmodule MyApp.UserService do
  alias MyApp.{Mailer, NotificationEmail}
  alias Ecto.Multi

  def insert(user_changeset) do
    Multi.new
    |> Multi.insert(:user, user_changeset)
    |> Multi.run(:notify_admin, &notify_admin/1)
  end

  defp notify_admin(%{user: user}) do
    user
    |> NotificationEmail.notify_admin
    |> Mailer.deliver
  end
end
```

I often name the service module functions after the operations they enhance, such as `insert`, `update`, or `delete`. Of course, there's nothing preventing you from using other names for more custom functionality.

As you can see in the example above, [Ecto.Multi][ecto-multi] allows us to encapsulate our business logic into a struct which can then be passed around, appended to, and run as a database transaction.

We can add arbitrary functions to the `Multi` operation, such as `notify_admin/1` which will cause the transaction to rollback if they return `{:error, ...}`. We can then call the whole thing with `Repo.transaction/1`:

```elixir
result =
  %User{}
  |> User.changeset(user_params)
  |> UserService.insert
  |> Repo.transaction

case result do
  {:ok, %{user: user}} ->
    # handle success
  {:error, :notify_admin, _failed_value, _changes_so_far} ->
    # handle the case that the email didn't send
  {:error, _failed_operation, _failed_value, _changes_so_far} ->
    # handle the generic error case
end
```

The error states are very detailed and allow you to be as specific or general as you want to be with your error handling.

Since each service function returns an [Ecto.Multi][ecto-multi], services can be chained together to execute as a single database transaction: [^2]

```elixir
user_operation = UserService.insert(user)
log_operation = LogService.insert(user)

user_operation
|> Multi.append(log_operation)
|> Repo.transaction
```

[Ecto.Multi][ecto-multi] structs are also inspectable using `Ecto.Multi.to_list/1`. This makes them pretty easy to unit test.

```elixir
user
|> UserService.insert
|> Multi.to_list
# => [
#   {:user, {:insert, user, []}}
#   {:notify_admin, {:run, fun, []}}
# ]
```

There are at least several other advantages to having a service layer built around [Ecto.Multi][ecto-multi]:

- Schemas can contain only pure functions, with no ActiveRecord bloat. All your external side effects can be kept in your service layer.
- Side-effects are opt-in, making testing **much easier**. Don't want all the side effects? Just `Repo.insert!` your data instead.
- Transactions can be automatically rolled back on failure, leaving no inconsistent mess behind. [^3]

Those are some pretty nice features for such a simple abstraction. Even so, you don't have to use [Ecto.Multi][ecto-multi] for everything, as we'll see in the next section.

## A Guide for Rails Developers

For `before_create` or `before_update` logic, update your schema's changeset function(s) rather than using a service. Usually, this type of business logic only affects data, so it's a natural fit for your changeset function.

For example, if we want to automatically promote or demote articles to our home page based on a `like_count` field, we could do that with changeset functions like so:

```elixir
def changeset(schema, params) do
  schema
  |> cast(params)
  |> promote_popular
end

defp promote_popular(changeset) do
  if get_field(changeset, :like_count) > 1000 do
    put_change(changeset, :promoted, true)
  else
    put_change(changeset, :promoted, false)
  end
end
```

Or if we wanted to automatically encrypt a user's password:

```elixir
def changeset(schema, params) do
  schema
  |> cast(params)
  |> validate_required([:password])
  |> validate_confirmation(:password)
  |> encrypt_password
end

defp encrypt_password(changeset) do
  password = get_change(changeset, :password)

  if password do
    encrypted = Comeonin.Bcrypt.hashpwsalt(password)
    changeset
    |> put_change(:encrypted_password, encrypted)
    |> delete_change(:password)
    |> delete_change(:password_confirmation)
  else
    changeset
  end
end
```

For `after_create` or `after_update` logic, use an [Ecto.Multi][ecto-multi] service. This logic usually has to do with other database tables or external side effects, and therefore is a good fit for a service module. The documentation provides a good example:

```elixir
defmodule UserService do
  alias Ecto.Multi
  import Ecto

  def password_reset(account, params) do
    Multi.new
    |> Multi.update(:account, Account.password_reset_changeset(account, params))
    |> Multi.insert(:log, Log.password_reset_changeset(account, params))
    |> Multi.delete_all(:sessions, assoc(account, :sessions))
  end
end
```

The only important convention for service modules is that they return an Ecto.Multi structs: what you name the modules and functions themselves is much less important. Do what makes sense for your application.

## Conclusion

I've been looking for a viable alternative to ActiveRecord callbacks and their associated bloat for a long time, and I finally feel like I've found it with [Ecto.Multi][ecto-multi]. Be sure to check out its documentation, and if you're interested, I've also covered its API in more detailed in [this episode of LearnPhoenix.tv][ecto-multi-episode].


[^1]: This is only _one_ available option. You could also share business logic with a `Plug` or a plain module, depending on your application.

[^2]: As long as every operation has a unique name.

[^3]: This automatic rollback behavior is also opt-in. If you don't want one of your operations to ever cause a rollback, just ensure it always returns `{:ok, ...}`.

[ecto-multi]: https://hexdocs.pm/ecto/Ecto.Multi.html
[ecto-multi-episode]: https://www.learnphoenix.tv/episodes/ecto-multi
[remove-callbacks]: https://github.com/elixir-ecto/ecto/issues/1104
[run/3]: https://hexdocs.pm/ecto/Ecto.Multi.html#run/3
[ecto-schema]: https://hexdocs.pm/ecto/Ecto.Schema.html
