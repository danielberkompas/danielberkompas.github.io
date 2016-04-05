---
layout: post
title: Manage Environment Variables in Elixir
author: Daniel Berkompas
categories:
  - elixir
date: 2015-03-21 07:00:00 -0800
---

I love how the Elixir build tool, `Mix`, has built-in support for configuration settings. It makes configuring packages much simpler by providing a standard interface for config settings.

I'm currently developing a [Twilio][twilio] API client for Elixir. While I develop and test it, I need to store an "Account SID" and "Auth token" to make requests. Naturally, I turned to Mix config.

<!-- more -->

Because I needed to have separate configuration files for each environment, so that I could test with different credentials, I set up `config/config.exs` like this:

```elixir
use Mix.Config

import_config "#{Mix.env}.exs"
```

I then created `config/development.exs` and `config/test.exs` files to store environment-specific settings.

```elixir
# config/development.exs
use Mix.Config

config :ex_twilio, account_sid: "SID HERE"
config :ex_twilio, auth_token:  "TOKEN HERE"
```

But I'm going to be pushing this project to Github when it's ready, so I don't want to store my real credentials here. Instead, I want to use `ENV` variables. So, I update my `config/<environment>.exs` files to look like this:

```elixir
use Mix.Config

config :ex_twilio, account_sid: System.get_env("TWILIO_ACCOUNT_SID")
config :ex_twilio, auth_token:  System.get_env("TWILIO_AUTH_TOKEN")
```

After trying a variety of different solutions, it seems like the best way currently to get these variables to be set is to create a `.env` file in your project directory:

```
export TWILIO_ACCOUNT_SID="<SID HERE>"
export TWILIO_AUTH_TOKEN="<TOKEN HERE>"
```

Then, simply run `source .env` in your shell before you run any `iex` or `mix` commands. Done!

[twilio]: http://twilio.com
