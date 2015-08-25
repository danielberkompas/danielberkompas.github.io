---
layout: "post"
title: "Encrypting Data With Ecto"
author: "Daniel Berkompas"
categories:
  - elixir
  - security
date: 2015-07-03 07:00:00 -0800
---

**Editor's Note: This post has been substantially updated since [it was first 
posted][original-post]. A much stronger crypto implementation has been used and
the code has been reworked to be cleaner and more efficient.**

In the future, as privacy becomes more and more of an issue, we're going to be
encrypting a lot more of the data we store on the web.  With that in mind, I
thought it would be a good idea to figure out a good way to integrate data
encryption with [Elixir's][elixir] database library, [Ecto][ecto].

<!-- more -->

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
  # Encrypts each plaintext with a different, random IV. This is much more
  # secure than reusing the same IV, and is highly recommended.
  def encrypt(plaintext) do
    iv    = :crypto.strong_rand_bytes(16) # Random IVs for each encryption
    state = :crypto.stream_init(:aes_ctr, key, iv)

    {_state, ciphertext} = :crypto.stream_encrypt(state, to_string(plaintext))
    iv <> ciphertext # Prepend IV to ciphertext, for easier decryption
  end

  def decrypt(ciphertext) do
    # Split the IV that was used off the front of the binary. It's the first
    # 16 bytes.
    <<iv::binary-16, ciphertext::binary>> = ciphertext
    state = :crypto.stream_init(:aes_ctr, key, iv)

    {_state, plaintext} = :crypto.stream_decrypt(state, ciphertext)
    plaintext
  end

  # Convenience function to get the application's configuration key.
  defp key do
    Application.get_env(:encryption, MyApp.AES)[:key]
  end
end
```

The module can then be used pretty simply:

```elixir
MyApp.AES.encrypt("hello world!")
|> MyApp.AES.decrypt
# => "hello world!"
```

You can configure the key to use in the `config.exs` for your app:

```elixir
config :my_app, MyApp.AES,
       key: :base64.decode("..."), # assuming your key is in base64
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
  import MyApp.AES

  # Assert that this module behaves like an Ecto.Type so that the compiler can
  # warn us if we forget to implement the 4 callback functions below.
  @behaviour Ecto.Type

  # This defines the base type of this kind of field in the database.
  def type, do: :binary

  # This is called on a value in queries if it is not a string.
  def cast(value) do
    {:ok, to_string(value)}
  end

  # This is called when the field value is about to be written to the database
  def dump(value) do
    ciphertext = value |> to_string |> encrypt
    {:ok, ciphertext}
  end

  # This is called when the field is loaded from the database
  def load(value) do
    {:ok, decrypt(value)}
  end
end
```

We're almost done! Now, since the encryptor we wrote operates directly on
binary, the fields we encrypt should be `:binary` fields in the database. 
Suppose we have an `Ecto.Model` with a binary name attribute like this:

```elixir
defmodule MyApp.User do
  use Ecto.Model

  schema "users" do
    field :name, :binary
  end
end
```

To encrypt the name field, you just need to specify the `MyApp.EncryptedField`
type instead of `:binary`:

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

Querying encrypted fields is difficult, because our encryptor is designed 
specifically _not_ to produce the same ciphertext twice for security reasons. 
To understand the difficulty, consider the following example.

- Suppose you have a user with the email address `test@example.com`. This value is
  encrypted in a binary field in the database.

- Now, someone comes along and tries to log in as `test@example.com`, and you want
  to query the database for a user with that email address. 

- You can't search for `test@example.com`, because that value doesn't exist in 
  the database. Neither can you run the email address through the encryptor, 
  because that will produce a different value than what is in the database.

So, how can you query for the email address? The answer is to **add another
field** to the database called `:email_hash`, which will contain a _hash_ of the
`:email` field's contents. This is both secure and convenient: secure because
the contents can't be reconstructed from a good hash; convenient, because hash
algorithms produce the same result for the same plaintext every time.

Let's implement another `Ecto.Type`, which will automatically hash the value of
a field using the recommended SHA256 algorithm:

```elixir
defmodule MyApp.HashField do
  @behaviour Ecto.Type

  def type, do: :binary

  def cast(value) do
    {:ok, to_string(value)}
  end

  def dump(value) do
    {:ok, hash(value)}
  end

  def load(value) do
    {:ok, value}
  end

  def hash(value) do
    :crypto.hash(:sha256, value)
  end
end
```

Then, we add this field to the schema:

```elixir
defmodule MyApp.User do
  use Ecto.Model

  schema "users" do
    field :name, MyApp.EncryptedField
    field :email, MyApp.EncryptedField
    field :email_hash, MyApp.HashField
  end

  # We must ensure that the email_hash field is always a hash of the same value
  # held in the email field, or queries will be inaccurate.
  before_insert :set_hashed_fields
  before_update :set_hashed_fields

  defp set_hashed_fields(changeset) do
    changeset
    |> put_change(:email_hash, get_field(changeset, :email))
  end
end
```

And now, we can easily query on the `:email_hash` field, because Ecto will
automatically use our `HashField` module to convert our search term to a hash:

```elixir
email = "test@example.com"

MyApp.Repo.get_by(MyApp.User, email_hash: email)
MyApp.Repo.one(from u in MyApp.User, where: u.email_hash == ^email)

# Both produce this query:
#
# SELECT u0."id", u0."name", u0."email", u0."email_hash", u0."inserted_at",
# u0."updated_at" FROM "users" AS u0 WHERE (u0."email_hash" = $1) [<<151, 61,
# 254, 70, 62, 200, 87, 133, 245, 249, 90, 245, 186, 57, 6, 238, 219, 45, 147, 
# 28, 36, 230, 152, 36, 168, 158, 166, 93, 186, 78, 129, 59>>]
```

And, we're done!

## Get the Code

All this code and more is over on a Phoenix sample project I put up on Github.
Here's [the relevant commit][commit]. It includes tests and more examples,
including how to validate the uniqueness of an encrypted field.

## Conclusion

We implemented the main features of [attr_encrypted][attr_encrypted], a somewhat
complicated Ruby gem, in about 60 lines of code, without any monkey 
patching! This implementation is also substantially more secure than
[attr_encrypted][attr_encrypted]'s default settings, which seem to rely on 
reusing IVs and using AES in CBC mode.

Even better, this solution is very easy to understand and is very extensible. 
For example, if you wanted to use a physical Hardware Security Module to do the
encryption and decryption, you could just write a custom encryptor and use it 
instead in your `EncryptedField` module.

I wasn't sure how easy this would be to implement, and I'm very happy with the
result. My confidence in Elixir as a tool continues to rise.

**READ THIS NEXT: [Changing Your Ecto Encryption Key][change-key]**

<hr />

**Credits**

Credit to [@victorluft](http://twitter.com/victorluft) for suggesting much
better crypto, and [@josevalim](http://twitter.com/josevalim) for suggesting
that I not make the encryptor a GenServer, since this could be a performance
bottleneck.

**Security Note**

Adding a hashed version of a field is a slight security risk, because it reveals
rows which have the same value in that field. Their hashes will match. Depending
on your threat model, this may mean you'll need a different solution for
querying on encrypted fields.

[change-key]: /elixir/security/2015/07/09/changing-your-ecto-encryption-key.html
[original-post]: https://github.com/danielberkompas/danielberkompas.github.io/blob/c6eb249e5019e782e891bfeb591bc75f084fd97c/_posts/2015-07-03-encrypting-data-with-ecto.md
[commit]: https://github.com/danielberkompas/phoenix_ecto_encryption_sample/commit/80c9b75a39f89a203f80617e2c00c062e9904217
[crypto]: http://www.erlang.org/doc/man/crypto.html
[elixir]: https://github.com/elixir-lang/elixir
[ecto]: https://github.com/elixir-lang/ecto
[ecto_type]: http://hexdocs.pm/ecto/Ecto.Type.html
[attr_encrypted]: https://github.com/attr-encrypted/attr_encrypted
