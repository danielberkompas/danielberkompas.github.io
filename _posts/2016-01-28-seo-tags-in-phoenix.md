---
layout: "post"
author: "Daniel Berkompas"
title: "SEO Tags In Phoenix"
comments: true
---

Public facing websites need to have some basic search engine optimization (SEO) tags, such as `<title>` and `<meta name="description">`. In Rails, you could achieve this pretty simply by putting a `yield :head` tag in the appropriate layout:

```erb
<head>
  <%= yield :head %>
</head>
```

Inside any of your views, you could then use `content_for` to populate this section:

```erb
<% content_for :head do %>
  <title>My Awesome Site</title>
  <meta name="description" content="..." />
<% end %>

<!-- More view content here... --->
```

This is nice, in that it keeps the HTML all together in more or less the same place. In [Phoenix][phoenix], this is actually a bit more difficult to do elegantly.

## Why It's Hard

In Phoenix, your layout is rendered _before_ your page template is rendered. Take this layout, for example:

```erb
<html>
  <head>
    <title></title>
    <meta name="description" content="" />
  </head>
  <body>
    <%= render @view_module, @view_template, assigns %>
  </body>
</html>
```

Because Phoenix renders templates as functions, by the time your `@view_template` is rendered, the `<head>` is already rendered. So, there's nothing that your view template can do to inject content back up into `<head>`. The `content_for` approach won't work.

I think there are at least four approaches to dealing with this: two of which I've seen in the wild, one which I came up with today, and one I used to use.

## 1. Assigns in Each Controller Action

The first approach modifies the layout to look like this:

```erb
<html>
  <head>
    <title><%= @title %></title>
    <meta name="description" content="<%= @meta %>" />
  </head>
  <body>
    <%= render @view_module, @view_template, assigns %>
  </body>
</html>
```

Then, in each controller action in the app, the developer must specify `title` and `meta` assigns:

```elixir
def index(conn, _params) do
  conn
  |> assign(:title, "My Awesome App")
  |> assign(:meta, "...")
  |> render("index.html")
end
```

There are a few downsides to this. First, the title and meta description can dominate the controller action, making it harder to see the business logic. Second, it can easily end up being pretty ugly if you don't put it all on one line:

```elixir
def index(conn, _params) do
  conn
  |> assign(:title, "My Awesome App")
  |> assign(:meta, """
     Lorem ipsum dolor sit amet, consectetur adipiscing elit. Aliquam feugiat 
     nibh ligula. Maecenas egestas nibh cursus erat sodales, vitae congue nisi
     tempus. Nam mattis et velit eu lacinia.
     """)
  |> render("index.html")
end
```

I think we can do better.

## 2. Delegated Functions

Another alternative I've seen is to delegate to functions on your view module, like this:

```erb
<html>
  <head>
    <title><%= @view_module.title(@view_template, assigns) %></title>
    <meta name="description" content="<%= @view_module.meta(@view_template, assigns) %>" />
  </head>
  <body>
    <%= render @view_module, @view_template, assigns %>
  </body>
</html>
```

Then, in your view module, you can define different titles or meta descriptions for different layouts, or set a default.

```elixir
defmodule MyApp.PageView do
  use MyApp.Web, :view
  
  def title("index.html", _assigns), do: "My Awesome App"
  def title(_other, _assigns), do: "Default Title"
  
  def meta("index.html", _assigns), do: "..."
  def meta(_other, _assigns), do: "Default Meta"
end
```

If you didn't want to have to define these functions on every view, you could create a mixin that would define default implementations of these `title` and `meta` functions.

```elixir
defmodule MyApp.SEO.Defaults do
  defmacro __using__(_) do
    quote do
      def title(_other, _assigns), do: "Default Title"
      def meta(_other, _assigns), do: "Default Meta"
      
      defoverridable [title: 2, meta: 2]
    end
  end
end
```

Then, `use` this module in your `web/web.ex` to make it affect all views.

```elixir
def view do
  quote do
    # ...
    use MyApp.SEO.Defaults
  end
end
```

This approach has the advantage of cleaning up the controller, but I think it's a bit less clear where the titles and meta descriptions are being generated.

## 3. Use a Plug

A huge number of problems can be solved with Plug, so today, I thought I would try to use it to solve this problem. It's pretty simple to create a custom plug for this:

```elixir
defmodule MyApp.SEO.Plug do
  def put_seo(%{private: %{phoenix_action: action}} = conn, settings) do
    settings = settings[action] || []

    conn
    |> assign(:title, settings[:title])
    |> assign(:meta, settings[:meta])
  end
end
```

The plug takes a "settings" argument, which we read from to determine what the `title` and `meta` assigns should be. I just include it in the `controller` section of `web/web.ex`:

```elixir
def controller do
  quote do
    # ...
    import MyApp.SEO.Plug
  end
end
```

And then I can use it in my controllers, like so:

```elixir
defmodule MyApp.PageController do
  use MyApp.Web, :controller
  
  @meta %{
    index: %{
      title: "My Awesome App",
      meta: """
      Lorem ipsum dolor sit amet, consectetur adipiscing elit. Aliquam feugiat 
      nibh ligula. Maecenas egestas nibh cursus erat sodales, vitae congue nisi
      tempus. Nam mattis et velit eu lacinia.
      """
    },
    contact: %{
      title: "Contact Us"
      meta: "..."
    }
  }
  
  plug :put_seo, @meta
  
  def index(conn, _params) do
    # ...
  end
  
  def contact(conn, _params) do
    # ...
  end
end
```

And my layout looks like this:

```erb
<html>
  <head>
    <title><%= assigns[:title] || "Default" %></title>
    <meta name="description" content="<%= assigns[:meta] || "Default" %>" />
  </head>
  <body>
    <%= render @view_module, @view_template, assigns %>
  </body>
</html>
```

While I'm not completely happy with it, (it would be nice to get all that text out of the controller), I think this is pretty clear. The controller actions are also uncluttered, which is a plus.

## 4. `render_existing/2`

Both Chris McCord and José Valim mentioned `render_existing/2` when they saw this post. I knew about it, (I believe it was one of my questions on IRC that prompted its creation, actually) but I don't really prefer it for this use case.

To use `render_existing` for this, you'd change your layout to look like this:

```erb
<html>
  <head>
    <%= render_existing @view_module, "meta." <> @view_template, assigns %>
  </head>
  <body>
    <%= render @view_module, @view_template, assigns %>
  </body>
</html>
```

The `render_existing` function will silently render nothing if the given template doesn't exist. You could then create a `meta.index.html.eex` file that contains whatever content you want:

```
<title>My Awesome App</title>
<meta name="description" content="..." />
```

This feels a bit more like Rails, but you end up with an extra file for each action in your controller, which feels a little odd.

```
meta.edit.html.eex
meta.index.html.eex
meta.new.html.eex
edit.html.eex
index.html.eex
new.html.eex
```

You can avoid this by defining these templates as functions in your view rather than template files:

```elixir
def render("meta.index.html", _assigns) do
  ~E{
    <title>My Awesome App</title>
    <meta type="description" content="..." />
  }
end
```

The `~E` sigil allows you to write EEX code inline in your view. This is fine, but it still can get a little messy looking with long meta descriptions, and you end up with a lot of function definitions.

I personally think this is likely to confuse the other devs on the projects I work on, so I've chosen not to establish it as the "standard" way at [Infinite Red](http://infinite.red). It is definitely a valid option though.

## Other Approaches

I'm pretty sure you could create a plug that used the [Gettext][gettext] API to extract all the strings out to .po files. I'm just not sure that's worth the added complexity and explaining to new developers.

It would be nice if Phoenix had an "official" recommendation on how to populate these SEO tags, but in the meantime, I hope these suggestions helped!

[phoenix]: https://phoenixframework.org
[gettext]: http://hexdocs.pm/gettext/Gettext.html
