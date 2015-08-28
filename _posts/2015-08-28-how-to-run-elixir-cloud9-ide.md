---
layout: "post"
title: "How to Run Elixir in Cloud9's IDE"
categories: 
  - elixir
---

[Cloud9](http://c9.io) is a great web-based development platform. If you
don't have access to a dedicated machine you can set up for development, or if
you just prefer to keep all your coding in neat, tiny VMs, Cloud9 could be just
what you're looking for. It's particularly good for students learning to code.

Cloud9 doesn't provide an Elixir-specific workspace template, so you have to
configure one yourself. Here's how to do that:

1. Create a Cloud9 account. If you already have a Github account, just log in
   with that.
2. Click "Create a new workspace".
3. Give it a name, say, "Elixir".
4. Select the "Custom" template.
5. When the workspace loads, run the following commands in the console:

```bash
# For some reason, installing Elixir tries to remove this file
# and if it doesn't exist, Elixir won't install. So, we create it.
sudo touch /etc/init.d/couchdb

# Standard Ubuntu Elixir installation instructions
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
sudo dpkg -i erlang-solutions_1.0_all.deb
sudo apt-get update
sudo apt-get install elixir
```

With that, you're ready to code! `iex`, `elixir`, and `mix` should all be
available, and Cloud9 has syntax coloring for Elixir code as well. Happy coding!
