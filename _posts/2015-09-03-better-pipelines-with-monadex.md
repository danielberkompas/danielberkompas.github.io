---
layout: "post"
title: "Better Pipelines with Monadex"
author: "Daniel Berkompas"
---

Before I get started, let me be blunt, **this is not another monad tutorial**. 
The world has enough of those already.  I'm not going to write about functors 
and applicatives or other technical theory.  This is just a post about how a 
particular monad made my life better.

## The Problem: Network Requests

I'm building a [Phoenix][phoenix] app where people can buy something. So, I 
needed to integrate with my favorite gateway, [Stripe][stripe], which involves 
making a series of network requests each time a user makes a purchase.

Specifically, I needed to do these things for each purchase:

1. Ensure the user hasn't already purchased the thing.
2. **[Network]** Create a Stripe Customer for the user, if it doesn't exist.
3. **[Network]** Create a Stripe Charge for the purchase amount.
4. Update the database to reflect the fact that the purchase has been made.

If any one of steps 1-3 fail, I don't want to perform the remaining steps.
Instead, the process should fail immediately and return the error.

## The Pipeline Operator Isn't Enough

I started out with a naive solution, using Elixir's normally adequate pipe 
operator to tie the steps together:

```elixir
result = user
         |> assert_not_purchased_yet
         |> create_stripe_customer(stripe_token) # From Stripe.js
         |> create_stripe_charge
         |> update_database
```

The obvious problem with this is that regardless of the result of each function,
the next function will be called. For example, `create_stripe_charge/1` will
still run even if `create_stripe_customer/2` failed.

I can work around this by making each function return either `{:ok, state}` or
`{:error, reason}`. If a function gets `{:error, reason}`, it should do nothing,
which would have the same result as if it wasn't called at all.

Here's how that looks:

```elixir
def create_stripe_customer({:ok, state}, stripe_token) do
  # Create Stripe customer, return {:ok, state} or {:error, reason}
end
def create_stripe_customer({:error, _} = error, _stripe_token) do
  error # just return the error
end

def create_stripe_charge({:ok, state}) do
  # Create stripe charge, return {:ok, state} or {:error, reason}
end
def create_stripe_charge({:error, _} = error) do
  error
end
```

This way, if `create_stripe_customer/2` returns an error, then
`create_stripe_charge/1` will do nothing. We can repeat this pattern all the way
down the chain, and pipeline will then look something like this:

```elixir
result = user
         |> assert_not_purchased_yet
         |> create_stripe_customer(stripe_token) # From Stripe.js
         |> create_stripe_charge
         |> update_database

case result do
  {:ok, state}     -> # Display success to user
  {:error, reason} -> # Display error reason to user
end
```

This works, but it isn't very elegant. I have to add a new function definition
for each function in the pipeline to handle the error case.

## Monads to the Rescue!

I knew enough about monads at this point to vaguely understand that they are a
bit like pipelines. Maybe there was a kind of monad that could make this better?

It turns out that there is. The excellent [Monadex][monadex] library for Elixir
provides just what I needed, the `Monad.Result` monad. Rather than talk theory,
let's just look at how we use this thing:

```elixir
defmodule MyApp.Purchase do
  use Monad.Operators # Brings in the ~>> bind operator

  # Import functions from the Monad.Result module.
  # These will be used to wrap the state that we pass through
  # all of our functions.
  import Monad.Result, only: [success?: 1,
                              unwrap!: 1,
                              success: 1,
                              error: 1]

  def create(user, stripe_token) do
    result = success(user) # Wrap user with the "success" monad state
             ~>> fn user -> assert_not_purchased_yet(user) end
             ~>> fn user -> create_stripe_customer(user, stripe_token) end
             ~>> fn user -> create_stripe_charge(user) end
             ~>> fn user -> update_database(user) end
             
    if success?(result) # %Monad.Result{type: :success, value: user}
      value = unwrap!(result) # Same as result.value
      # Display success to user
    else
      # Display error to user
    end
  end

  # ...
end
```

First, we `use` the `Monad.Operators` module. This brings in the `~>>` operator
which we'll get to in a minute. Next, we import most of the functions from
`Monad.Result`.

Underneath the hood, a `Monad.Result` is just a struct that wraps state, not all
that different from a `{:ok, state}` or `{:error, reason}` tuple.

```elixir
%Monad.Result{type: :error | :success, value: state}
```

So, the first thing we do is wrap our state, the `user` variable, with a
`Monad.Result` struct. Since this is the first part of our pipeline, we'll start
with a `:success` struct.

```elixir
success(user) # => %Monad.Result{type: :success, value: user}
```

The `~>>` operator, when used with the `Monad.Result` monad, will ensure that
the next function in the pipeline only runs if the previous one returned a 
`:success` result. Otherwise, it terminates immediately and returns the last 
`Monad.Result` that was returned.

If we use a `~>>` pipeline instead of the regular `|>` pipeline, our functions
can then look like this:

```elixir
def assert_not_purchased_yet(user) do
  case purchased?(user) do
    true  -> error("Already purchased.")
    false -> success(user) # Return whatever state the next function needs
  end
end

def create_stripe_customer(%{stripe_customer_id: id}) when id != nil do
  success(user) # The user already has a stripe customer id, so do nothing
end
def create_stripe_customer(user, stripe_token) do
  case Stripe.Customer.create(user) do # pseudocode
    {:ok, _customer} -> success(user)
    {:error, reason} -> error(reason)
  end
end

def create_stripe_charge(user) do
  case Stripe.Charge.create(...) do # pseudocode
    {:ok, _charge}   -> success(user)
    {:error, reason} -> error(reason)
  end
end

# etc ...
```

Much cleaner! Now, we'll only make the Stripe network calls if the previous step
was successful. There's no nested tree of `if` statements, just a simple
pipeline. And we can operate on the result of all these operations very simply:

```elixir
if success?(result) do
  # Get the value out of the monad
  value = unwrap!(result)
  # render success
else
  # render error
end
```

## Errors Prevented

This implementation elegantly handles each of the following situations:

- The user has already purchased the product.
- The user is already associated with a Stripe customer.
- The Stripe customer could not be created.
- The Stripe customer could be created but the charge could not.

Further, it only does as much work as necessary to determine the result. It
fails fast, allowing the user to get feedback as soon as possible.

## Conclusion

This is the first time I understood how a monad would help me, and I hope it was
useful to you too! Whenever you find yourself wishing for a better type of
pipeline, give [Monadex][monadex] a try.

[phoenix]: http://phoenixframework.org
[stripe]: http://stripe.com
[monadex]: https://github.com/rob-brown/MonadEx
