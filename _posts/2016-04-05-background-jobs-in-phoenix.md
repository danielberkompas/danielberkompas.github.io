---
layout: "post"
author: "Daniel Berkompas"
title: "Background Jobs in Phoenix"
comments: true
---

In Ruby on Rails, it's very common to use background worker libraries like [Sidekiq][sidekiq] to speed up requests and do work asynchronously. Rather than doing all the work that needs to be done inline (and blocking other requests), a background job can speed things up significantly. Sidekiq is great, and you should definitely use it in your Ruby projects.

However, because Phoenix is built with Elixir, we can get pretty far with just the built-in concurrency primitives and OTP, without any outside libraries.

## [Task.start_link](http://elixir-lang.org/docs/stable/elixir/Task.html)

If you don't care about whether the background job succeeds or not, and you don't need to get the result back, you can use `Task.start_link/1`. Simply pass an anonymous function to the `start_link/1` function, and it will be run in a background process.

```elixir
Task.start_link(fn -> IO.puts "Hello, World" end)
```

## [Task.async](https://elixir-lang.org/docs/stable/elixir/Task.html)

If you care about the result, then you should use the `Task` module's `async/1` function. Here too, you can pass an anonymous function to `async`, and it will be run in a new, separate process.

```elixir
Task.async fn ->
  do_something_in_the_background
end
```

Alternatively, you can give a module name, function name, and the function's arguments, instead of an anonymous function. To retrieve the result of the function once it finishes running, you wait for it using `Task.await/1`.

```elixir
# The `task` that is returned here is a struct, containing a reference
# to the process that was started
task = Task.async(ModuleName, :function, [args])

# `Task.await/1` will wait for a limited time for the background process
# to complete and return the result. If the background job doesn't
# complete within the time, an exception will be thrown.
result = Task.await(task)

# `result` is now bound to whatever value ModuleName.function(args) returns
```

This alone gets you a pretty long way, but there are situations where more is required.

## [Task.Supervisor](http://elixir-lang.org/docs/stable/elixir/GenServer.html)

If you have tasks that can be safely retried, you can use a `Task.Supervisor` to manage them. Just add the supervisor to your app's supervision tree in `lib/myapp.ex`.

```diff
children = [
  supervisor(MyApp.Endpoint, []),
  supervisor(MyApp.Repo, []),
+ supervisor(Task.Supervisor, [[name: MyApp.TaskSupervisor, restart: :transient]])
]
```

The `restart: :transient` option is important, because it causes the `Task.Supervisor` to retry failed tasks. The default setting is `:temporary`, and does not retry tasks.

With this in place, you can start a supervised task like so:

```elixir
Task.Supervisor.start_child MyApp.TaskSupervisor, fn ->
  do_background_work
end
```

Or, if you care about the result, you can use `Task.Supervisor.async/2` and await the results:

```elixir
task = Task.Supervisor.async MyApp.TaskSupervisor, fn ->
  do_background_work
end

result = Task.await(task)
```

If the tasks managed by this supervisor fail, they will be retried automatically until they succeed. It's also possible to start tasks on a separate node (separate hardware!):

```elixir
Task.Supervisor.async({MyApp.TaskSupervisor, :remote@local},
                      ModuleName, :function, [args])
```

This only works if the other node has a supervisor named `MyApp.TaskSupervisor` and is also running a version of `ModuleName`, of course.

## When are Libraries Needed?

You may still need a library if you need queues and job prioritization, or if your nodes don't have network access to each other, and you therefore have to share a queue through Redis. However, you can get pretty far using only the `Task` module.

[sidekiq]: http://sidekiq.org/
