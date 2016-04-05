---
layout: "post"
author: "Daniel Berkompas"
title: "GenServers as Concurrent Objects"
categories:
  - elixir
date: 2015-08-21 07:00:00 -0800
---

This is a post for fellow object-oriented developers trying to get their heads around how Elixir/Erlang use processes as a basic abstraction, rather than classes and objects.

<!-- more -->

## A Sample Object

In an object oriented language, such as Ruby, we might implement a `BankAccount` class like this:

```ruby
class BankAccount
  attr_reader :balance

  def initialize(starting_balance)
    @balance = starting_balance
  end

  def deposit(amount)
    @balance += amount
  end

  def withdraw(amount)
    @balance -= amount
  end
end
```

This class could then be used like this:

```ruby
account = BankAccount.new(0.0)
account.deposit(50.0)
account.withdraw(25.0)
account.balance # => 25.0
```

## A GenServer Equivalent

In Elixir, we can implement a `GenServer` that behaves very similarly:

```elixir
# Create a new GenServer process with an initial state (balance) of 0, using the
# BankAccount module to process messages to the process. Returns a process ID.
{:ok, account} = GenServer.start(BankAccount, [0.0])

# Send messages to the process to deposit or withdraw amounts.
# cast/2 runs asynchronously and doesn't wait for a response.
GenServer.cast(account, {:deposit, 50.0})
GenServer.cast(account, {:withdraw, 25.0})

# Query the process for the balance, and waits for a response.
GenServer.call(account, :balance) # => 25.0
```

`GenServer` here will fire callbacks on the `BankAccount` module in response to the messages sent to the process.

```elixir
defmodule BankAccount do
  use GenServer

  # Casts don't reply to the caller, but simply modify the state
  def handle_cast({:deposit, amount}, balance) do
    {:noreply, balance + amount}
  end

  def handle_cast({:withdraw, amount}, balance) do
    {:noreply, balance - amount}
  end

  # Calls return a value, as well as the modified state
  def handle_call(:balance, balance) do
    {:reply, balance, balance}
  end
end
```

We can hide all the GenServer implementation behind a public API on the `BankAccount` module. (Not shown) Our code will then look very similar to Ruby's:

```elixir
{:ok, account} = BankAccount.start(0.0)
BankAccount.deposit(account, 50.0)
BankAccount.withdraw(account, 25.0)
BankAccount.balance(account) # => 25.0
```

## GenServer "Singletons"

It is possible to refer to `GenServer` processes by a name rather than by a process ID. In this case, a `GenServer` behaves more like a singleton rather than an object instance, because there is only one process running.

This naming is accomplished by passing the `:name` option to `GenServer.start`:

```elixir
GenServer.start(BankAccount, [0.0], name: BankAccount)
```

You can then send messages to the `BankAccount` process using its name:

```elixir
GenServer.cast(BankAccount, {:deposit, 50.0})
```

If we reimplemented our `BankAccount` module as a named GenServer, (not shown) we could use it like this:

```elixir
# In our application initialization
BankAccount.start([0.0])

# Then, anywhere in our code:
BankAccount.deposit(50.0)
BankAccount.withdraw(25.0)
BankAccount.balance # => 25.0
```

This `BankAccount` process can then be called from other nodes (think, microservices) in your cluster like so:

```elixir
GenServer.call({BankAccount, :accounts@localhost}, :balance) # => 25.0
```

No JSON APIs required!

## Contrasts

We see then that `GenServer` modules can behave very much like objects because they are a construct that both holds state and can perform operations on that state. However, they are different from objects in Ruby in a few important ways:

- A `GenServer` can hold only one value as its state. In Ruby, you can have as many instance variables as you like. However, this isn't a big limitation, since the state you store in a `GenServer` can be as complex as you want.

- `GenServer` operations are automatically spread across your CPU cores. If the operation doesn't require a response, then the caller won't be blocked. However, it's important to keep in mind that a `GenServer` process can only do one thing at a time.

- Named `GenServer` processes can be called from other computers in the cluster, as shown above.

- `GenServer` processes shouldn't really be used as often as objects or in the same ways. They are designed for concurrency, not managing data structures like objects are. I compare them to objects just to relate them to something familiar.

## Conclusion

I know that it was a real "lightbulb" moment for me when I realized that GenServers are Elixir/Erlang's answer to objects. Hopefully you've found this article helpful! If something isn't clear, ask me a question [on Twitter](http://twitter.com/dberkom) and I'll see if I can clarify the post.
