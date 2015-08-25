---
layout: "post"
author: "Daniel Berkompas"
title: "How to Write Guard Macros"
categories:
  - elixir
date: 2015-06-10 07:00:00 -0800
---

I recently discovered that it is possible to write custom guard macros for
Elixir, provided that the macro expands to expressions that are supported in
guards natively.

I used this to create an `is_blank` guard. Elixir doesn't come with a `blank?`
function like Ruby does, so you have to do it manually. Blank values are `" "`,
`""`, and `nil`. To check `blank?` in Elixir, you can check if a given value is
`in` this array of blank values.

<!-- more -->

```elixir
value in [" ", "", nil] # => true / false
```

You can do this in a guard:

```elixir
def foo(bar) when bar in [" ", "", nil] do
  # baz ...
end
```

Using `in` directly works great if you only have to do it once. But if you find
yourself wanting `is_blank` all over your code, you can write a macro like so:

```elixir
defmacro is_blank(value) do
  quote do
    unquote(value) in [" ", "", nil]
  end
end
```

You can then use `is_blank` in your guard statement, because the `in` clause is
supported by guards natively:

```elixir
def foo(bar) when is_blank(bar) do
  # baz ...
end
```

You can also use `is_blank` like any other function in other parts of your code,
outside of guards.

## Gotchas

- Macros used in guards must be defined in a **different module** than the one
  where they are being used. This is due to the way Elixir compiles macros.

- You must `require` or `import` your other module in order to be able to use
  the Macro. If you `require`, you'll have to use the macro like this:
  `OtherModule.macro`. If you use `import`, you can just use `macro`.
