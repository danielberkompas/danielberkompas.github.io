---
layout: "post"
title: "Encrypting Data With Ecto"
author: "Daniel Berkompas"
categories:
  - elixir
  - security
---

In the future, as privacy becomes more and more of an issue, we're going to be
encrypting a lot more of the data we store on the web.  With that in mind, I
thought it would be a good idea to figure out a good way to integrate data
encryption with [Elixir's][elixir] database library, [Ecto][ecto].

## Requirements

In Rails, we have a gem called [attr_encrypted][attr_encrypted] which makes it
easy to have encrypted attributes on ActiveRecord models. The important
features are:

- Transparent encryption/decryption of fields
- Custom encryptors, allowing for customizable security
- Automatic query support for encrypted fields

Let's take a look at how to replicate this in [Ecto][ecto].

## Building Our Encryptor

Before we can encrypt anything, we're going to need to create a module to handle
encryption and decryption. Erlang comes with a good [crypto][crypto] module,
which will serve as our base.

For this use case, I've chosen to use AES encryption in CTR mode, but you could
just as easily use any other type of encryption supported by [crypto][crypto].

```elixir
defmodule MyApp.AES do
  # Since this module will hold configuration state and perform operations,
  # it should be a GenServer, and run as its own process.
  use GenServer

  # Start the server process as a named process, so that we don't have to
  # worry about pids.
  def start_link(options) do
    GenServer.start_link(__MODULE__, options, name: __MODULE__)
  end

  # This is called by GenServer after `start_link`, and configures what state
  # the server will start out with.
  def init(options) do
    # AES in CTR mode is a stream cipher, and requires state to be set up before
    # it can perform any operations. Here, we set up the state by specifying the
    # mode, the AES key, and an initialization vector.
    state = :crypto.stream_init(:aes_ctr, options[:key], options[:iv])

    # We then return the crypto configuration as the initial GenServer state.
    {:ok, state}
  end

  # Public wrapper function for encryption. Ensures that the value is a string 
  # before trying to encrypt it.
  def encrypt(string) do
    string = to_string(string)
    GenServer.call(__MODULE__, {:encrypt, string})
  end

  # Public wrapper function for decryption.
  def decrypt(string) do
    GenServer.call(__MODULE__, {:decrypt, string})
  end

  # Server callback for encryption. Uses `crypto:stream_encrypt/2`, using the
  # crypto state saved in the GenServer process.
  def handle_call({:encrypt, string}, _from, state) do
    {_state, cipher} = :crypto.stream_encrypt(state, string)

    # The ciphertext is returned in base64 encoding to make it easier to store
    # in a regular database string field.
    {:reply, :base64.encode(cipher), state}
  end

  # Server callback for decryption. Uses `crypto:stream_decrypt/2`, using the
  # crypto state saved in the GenServer process.
  def handle_call({:decrypt, string}, _from, state) do
    # Assume input string is in base64
    string = :base64.decode(string)
    {_state, plaintext} = :crypto.stream_decrypt(state, string)
    {:reply, plaintext, state}
  end
end
```

The module can then be used pretty simply:

```elixir
{:ok, _pid} = MyApp.AES.start_link(key: "...", iv: "...")

MyApp.AES.encrypt("hello world!")
|> MyApp.AES.decrypt
# => "hello world!"
```

You can set up your app's supervisor to automatically start and maintain the 
`MyApp.AES` process:

```elixir
worker(OnSale.AES, [Application.get_env(:my_app, MyApp.AES)])
```

And configure the `:key` and `:iv` in your `config.exs`:

```elixir
config :my_app, MyApp.AES,
       key: :base64.decode("..."),
       iv: "..."
```

Now that we have an encryptor, we can look at integrating it with Ecto.

## Ecto.Type

To implement transparent encryption and decryption of fields, we need to add a
layer of code in Ecto's row insertion and loading logic, so that we can encrypt
the fields on save, and decrypt them when they are read. Fortunately, Ecto has
exactly what we need in [Ecto.Type][ecto_type].

[Ecto.Type][ecto_type] lets you define custom field types for Ecto's `schema`,
allowing you to modify the value of a field when it is loaded or saved. Here's a
custom `EncryptedField` type:

```elixir
defmodule MyApp.EncryptedField do
  # Assert that this module behaves like an Ecto.Type so that the compiler can
  # warn us if we forget to implement the 4 callback functions below.
  @behaviour Ecto.Type

  # This defines the base type of this kind of field in the database.
  def type, do: :string

  # This is called on a value in queries if it is not a string.
  def cast(value) do
    {:ok, to_string(value)}
  end

  # This is called when the field value is about to be written to the database
  def dump(value) do
    value = to_string(value)
    {:ok, OnSale.AES.encrypt(value)}
  end

  # This is called when the field is loaded from the database
  def load(value) do
    {:ok, OnSale.AES.decrypt(value)}
  end
end
```

We're almost done! Now, to encrypt a string field, all we have to do is change
it's type in the database. Suppose we have an `Ecto.Model` with a name attribute
like this:

```elixir
defmodule MyApp.User do
  use Ecto.Model

  schema "users" do
    field :name, :string
  end
end
```

To encrypt the name field, you just need to specify the `MyApp.EncryptedField`
type instead of `:string`:

```elixir
defmodule MyApp.User do
  use Ecto.Model

  schema "users" do
    field :name, MyApp.EncryptedField
  end
end
```

That's it! The field will be transparently encrypted and decrypted as the model
struct is saved to or loaded from the database.

## Querying

In order to query on an encrypted field, you need to encrypt the search term
before executing your SQL query. So, to find a user with the `:name` "Daniel",
you'd need to encrypt "Daniel" first, and then look for users where the `:name`
matches the encrypted value. For example:

```sql
SELECT * FROM users WHERE name = "ATQd64as";
```

The cool thing is, you _don't have to do anything more_ to get this
functionality in most cases! `Ecto.Repo` queries will automatically do this for
you:

```elixir
MyApp.Repo.get_by(MyApp.User, name: "Daniel")
# => SELECT u0."id", u0."name", 
#    FROM "users" AS u0 
#    WHERE (u0."name" = $1) ["ATQd64as"] (2.0ms)
```

However, you'll still need to encrypt values in your custom Ecto queries:

```elixir
name = MyApp.AES.encrypt(name)
from u in MyApp.User, where: u.name == ^name
```

## Conclusion

We implemented the main features of [attr_encrypted][attr_encrypted], a large
Ruby gem, in about 55 lines of code, without any monkey patching! This speaks to
how well Elixir and Ecto are designed.

This solution is very easy to understand and is very extensible. For example, if
you wanted to use a physical Hardware Security Module to do the encryption and
decryption, you could just write a custom encryptor and use it instead in your
`EncryptedField` module.

I wasn't sure how easy this would be to implement, and I'm very happy with the
result. My confidence in Elixir as a tool continues to rise.

[crypto]: http://www.erlang.org/doc/man/crypto.html
[elixir]: https://github.com/elixir-lang/elixir
[ecto]: https://github.com/elixir-lang/ecto
[ecto_type]: http://hexdocs.pm/ecto/Ecto.Type.html
[attr_encrypted]: https://github.com/attr-encrypted/attr_encrypted
