---
layout: "post"
author: "Daniel Berkompas"
title: "Reusable Templates in Phoenix"
comments: true
---

If you've spent any time with [React](https://facebook.github.io/react/) or its [look-a-likes](https://github.com/developit/preact), you've probably realized that most web apps have a lot of duplication in their templates. This is particularly true of server-rendered apps.

<!-- more -->

Templates should be organized as a set of reusable components, like this:

```html
<Tabs>
  <Tab name="All Products" />
  <Tab name="Featured" />
</Tabs>
```

This way, if you ever decide to change the markup for `<Tab>` or `<Tabs>`, you only have to change one place, and all the `<Tabs>` across your project will be updated automatically.

However, just because server-rendered apps aren't usually organized this way _doesn't mean they can't be._ In Phoenix, it's actually quite easy. Here's all you need to do.

1) Create a `ComponentView` and a `templates/component` directory:

```elixir
defmodule MyApp.Web.ComponentView do
  use MyApp.Web, :view
end
```

2) Define templates for each component you want to have. For example:

```erb
<!-- templates/component/tabs.html -->
<ul class="tabs">
  <%= assigns[:do] %>
</ul>

<!-- templates/component/tab.html -->
<li class="tab"><%= assigns[:name] %></li>
```

3) Use these templates whenever you want tabs:

```erb
<%= render ComponentView, "tabs.html" do %>
  <%= render ComponentView, "tab.html", name: "All Products" %>
  <%= render ComponentView, "tab.html", name: "Featured" %>
<% end %>
```

So far, so good, but it's too verbose. Let's add a `ComponentHelpers` module like so:

```elixir
# views/component_helpers.ex
defmodule MyApp.Web.ComponentHelpers do
  def component(template, assigns) do
    MyApp.Web.ComponentView.render(template, assigns)
  end
  
  def component(template, assigns, do: block) do
    MyApp.Web.ComponentView.render(template, Keyword.merge(assigns, [do: block]))
  end
end
```

Then, import it into your `view` function in the `MyApp.Web` module:

```elixir
def view do
  quote do
    # ...
    import MyApp.Web.ComponentHelpers
  end
end
```

Your markup can then be much more concise:

```erb
<%= component "tabs.html" do %>
  <%= component "tab.html", name: "All Products" %>
  <%= component "tab.html", name: "Featured" %>
<% end %>
```

There's no reason you couldn't shorten it up even more if you wanted:

```erb
<%= c "tabs.html" do %>
  <%= c "tab.html", name: "All Products" %>
  <%= c "tab.html", name: "Featured" %>
<% end %>
```

It's still not as concise as React, but I think it's close enough.

```html
<Tabs>
  <Tab name="All Products" />
  <Tab name="Featured" />
</Tabs>
```

Obviously, this is a contrived example with very simple markup, so it might seem like too much effort. However, I think it's a very useful approach whenever you're dealing with complicated markup that you need to reuse often.

