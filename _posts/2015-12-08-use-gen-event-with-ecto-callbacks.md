---
layout: "post"
title: "Using GenEvent With Ecto Callbacks"
author: "Daniel Berkompas"
---

Callbacks. It's common to write _tons_ of callback methods in Ruby ActiveRecord models, and they're one reason ActiveRecord models tend to end up so complicated.

Ecto also has callback functions such as `before_insert`, `before_update` and so on. In my most recent Elixir project, I didn't want to have my Ecto models end up as cluttered as my ActiveRecord models did.

## Eliminating Callbacks

The most obvious way to eliminate callbacks in your models is simply not to use them at all. Every time you need to perform an action `before_insert` or `after_update`, you simply do that action.

So, in your controller, if you need to send an email after a user is created, you could do:

```elixir
def create(conn, %{"user" => user_params}) do
  changeset = User.changeset(%User{}, user_params)

  case Repo.insert(changeset) do
    {:ok, user} ->
      send_email(user)
      # ...
    {:error, changeset} ->
      # ...
  end
end
```

However, this can become fragile if you start creating users in _multiple_ places. You have to remember to add the `send_email/1` call in each place. So, for this sort of thing, you really _need_ an `after_insert` callback function to do it reliably after every user is inserted.

But then we're back where we started, with the threat of an ever-expanding set of callback functions polluting our Ecto models.

## Use GenEvent

The solution I came up with for this is very simple. We can think of the model's lifecycle as a series of events: `insert`, `update` and `delete`. Therefore, we could use OTP's built-in event management feature to handle this: `GenEvent`.

First, set up an Event broadcaster, like so:

```elixir
defmodule MyApp.Event do
  @handlers [
    Footprint.Handlers.ModelHandler
  ]

  def start_link do
    {:ok, manager} = GenEvent.start_link(name: __MODULE__)
    Enum.each(@handlers, &GenEvent.add_handler(manager, &1, []))
    {:ok, manager}
  end

  def broadcast(event) do
    GenEvent.notify(__MODULE__, event)
  end
end
```

We're setting up a publish/subscribe model, and so the `@handlers` attribute is a list of subscriber modules. `start_link/0` will be used to start the event broadcaster, and add register all of the subscribers. The `broadcast/1` function will notify all subscribers of a new event.

We then add `MyApp.Event` to our supervisor to ensure it starts up:

```elixir
children = [
  worker(MyApp.Event, [])
]
```

### Broadcasting All Lifecyle Events

We now will create a module which will automatically broadcast lifecyle events. It looks like this:

```elixir
defmodule MyApp.ModelLifecycle do
  alias MyApp.Event

  defmacro __using__(_) do
    quote do
      import unquote(__MODULE__)

      after_insert :broadcast_event, [:insert]
      after_update :broadcast_event, [:update]
      after_delete :broadcast_event, [:delete]
    end
  end

  # Broadcasts events in this format:
  # {:model, :update, changeset}
  def broadcast_event(changeset, type) do
    Event.broadcast({:model, type, changeset})
    changeset
  end
end
```

The `__using__` macro imports the module and inserts all of the needed callbacks, pointing at `broadcast_event/2`, which then broadcasts that event on `MyApp.Event`, notifying all subscribers.

We can then `use` this `ModelLifecyle` module in our model:

```elixir
defmodule MyApp.User do
  use Ecto.Model
  use MyApp.ModelLifecyle

  # ...
end
```

Now, our `User` model will automatically broadcast each lifecyle event after it occurs. Using the GenEvent API, we can _subscribe_ to those notifications and do something with them.

For example, we could write an `EmailSubscriber`, like so:

```elixir
defmodule MyApp.EmailSubscriber do
  use GenEvent

  alias MyApp.User

  def handle_event({:model, :insert, %{model: %User{}} = changeset}, state) do
    send_email(changeset.model)
    {:ok, state}
  end
end
```

With a little tender love and care, (not shown) we can pretty up this API to look like so:

```elixir
defmodule MyApp.EmailSubscriber do
  use GenEvent

  alias MyApp.User

  def handle_insert(%User{} = user) do
    send_email(user)
  end

  # ...
end
```

## Benefits

This approach provides a few benefits:

- We can have multiple subscribers to the same events
- New subscribers require no changes to the model
- Event handling logic for multiple models can live in one place
- Lifecycle events are handled in a separate process, and therefore don't prevent your model from being saved if they fail. This can be a good or bad thing depending on your use case.

## Downsides

Like everything, this approach also has its downsides.

- Added complexity
- Implicit behaviour: it isn't immediately obvious how the logic for certain events is triggered. You need to know about the pub/sub behaviour to know where to look.

If either of these downsides is too much for you, you can also reference a foreign module in an Ecto callback, like so:

```elixir
after_insert OtherModule, :function_name
```

This allows you to delegate callback work to other modules, keeping your model clean, but requires you to change each model if you want to add another "subscriber".

## Conclusion

I hope you found this as interesting as I did when I discovered it. Hopefully I'll have more time for blog posts once my [LearnElixir.tv](https://www.learnelixir.tv) screencast series is finished.
