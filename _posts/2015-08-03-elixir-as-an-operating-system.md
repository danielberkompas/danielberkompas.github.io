---
layout: "post"
author: "Daniel Berkompas"
title: "Elixir as an Operating System"
categories:
  - elixir
---

I recently was using a [Synology Diskstation][synology], and I was very
impressed by their web admin interface. They have successfully emulated a 
desktop operating system, complete with downloadable programs, file browsing,
and more. You can manage the whole system from the browser in a way that feels
very much like Windows 7.

Even more impressively, Diskstations can update themselves and their apps with
one click, and sometimes, with no clicks at all. It's a remarkable achievement,
and it makes their products great.

However I suspect all is not well. Perhaps its just cynicism on my part, but I 
think the code underlying all of this is probably a mess. I don't know if it's 
PHP or not, but given the vast number of features that Synology offers and the 
fact that most mass-market software is a mess underneath the surface, I doubt 
that Synology is an exception. In other words, I doubt that I would enjoy 
_writing_ an app for a Diskstation as much as I might enjoy _using_ one.

Wordpress is somewhat similar to Synology, in that it provides a core package
which can then be extended by various plugins. It now also has automatic
updates, and maybe even automatic plugin updates. (I haven't checked for a 
while) But, as every developer who has ventured into the dark heart of Wordpress
knows, it isn't very well built at bottom. That's putting it _very_ mildly.

Developers hate working with it, and it costs the poor saps who use it boatloads
of money whenever they want to do something remotely custom. Wordpress sites are
buggy and prone to security issues, and are very difficult to administer due to
its convoluted UI, made worse by still more convoluted plugin UIs. Yet, 
Wordpress is still the most extensible and user-friendly website creation tool.

My experience with Diskstation got me thinking, could Elixir be used to build
a Wordpress replacement? Something that mere mortals could use and customize,
but which wasn't terrible underneath the surface? Something that wouldn't cause
developers everywhere to cringe when its name is mentioned? I think such a 
thing might be possible.

## Elixir's Advantages

Elixir has several advantages over PHP for such a project:

- It's a better language. It is more explicit, more consistent, and has better
  developer tools. Documentation (essential for plugins!) is easy to write and
  HTML docs can be generated easily.

- Plugins == OTP Apps. Elixir has the concept of "apps" (persistent processes 
  that are supervised and automatically restarted) built right in. Plugins could
  easily be written as OTP apps. Elixir already "thinks" like an operating 
  system.

- Hot code swapping. Elixir supports upgrading a process to new code while the
  process stays running. This makes the challenge of one-click installs and
  upgrades a little easier.

- High availability. Elixir can maintain nearly zero-downtime through
  supervision trees, and plugins can maintain their own supervision trees.

- High performance. Budget conscious website creators could squeeze a lot of
  performance out of low end servers.

## Elixir's Disadvantages

Elixir also has a couple disadvantages:

- Installation could be more complicated. Erlang and Elixir are not as common
  as PHP, and would have to be present on the server for the system to run. Many
  web hosting companies offer a "Wordpress" button to install Wordpress on their
  servers, while nothing similar would exist for Elixir.

- Elixir is not a mainstream language. This would mean that for a while at
  least, very few plugins would be available.

## Worth a Shot?

I'm intrigued by the possibility of building a Wordpress replacement in Elixir. 
It really does seem like Elixir could be an ideal fit for this kind of thing.

However, rebuilding Wordpress in any language would certainly be no easy task, 
even with the helps that Elixir provides. If I get the chance, I may hack on it
for a few evenings and see what happens, and I'll definitely post about it.

If anyone else out there is inspired by this post and decides to steal the idea,
go ahead! Just let me know where you're hosting the project so I can contribute.

[synology]: https://www.synology.com/
