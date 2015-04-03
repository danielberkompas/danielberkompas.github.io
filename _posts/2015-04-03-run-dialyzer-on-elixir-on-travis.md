---
layout: post
title: "Run Dialyzer on Elixir on Travis CI"
categories:
  - elixir
---

In [my last post][contracts-gem], I talked about Elixir's typespec annotations
and Erlang's static analysis tool, [Dialyzer][dialyzer]. All that talk was great
and all, but how do you actually _use_ Dialyzer on Elixir projects?

In particular, how do you run Dialyzer on an Elixir project on Travis CI? In
today's blog post, I'll show you how.

## Persistent Lookup Table

Dialyzer requires a "persistent lookup table", or PLT, in order to perform its 
analysis.  This PLT holds references to all the system Erlang libraries, and 
Elixir's source code itself. In order to run Dialyzer on an Elixir project, 
you've got to first generate a PLT for your system.

Yes, the PLT has to be generated _for the system that you're going to run
Dialyzer on_, because the paths in the PLT are _hard coded_ when it is generated.
A PLT generated on your Mac won't work on Travis.

What's more, the PLT takes quite a while to generate. If you generated it every
time you ran your build on Travis, you'd slow down your builds by minutes.

Luckily for you, I've built a bunch of PLTs for Elixir on Travis already. You 
can find them here: [danielberkompas/travis_elixir_plts][plts]. You're welcome.

## .travis.yml

With a prebuilt PLT, running dialyzer on Travis is simple:

```yaml
language: elixir
otp_release:
  - 17.4
before_script:
  # Set download location
  - export PLT_FILENAME=elixir-${TRAVIS_ELIXIR_VERSION}_${TRAVIS_OTP_RELEASE}.plt
  - export PLT_LOCATION=/home/travis/$PLT_FILENAME
  # Download PLT from danielberkompas/travis_elixir_plts on Github
  # Store in $PLT_LOCATION
  - wget -O $PLT_LOCATION https://raw.github.com/danielberkompas/travis_elixir_plts/master/$PLT_FILENAME
script:
  - mix test
  - dialyzer --no_check_plt --plt $PLT_LOCATION --no_native _build/test/lib/$YOUR_PROJECT_NAME/ebin
```

A few notes on the options passed to Dialyzer here:

- `--no_check_plt`: Increases speed. Dialyzer assumes PLT is up to date.
- `--no_native`: Also increases speed. Because it is used to analyze large
  projects, Dialyzer likes to compile parts into native code before doing
  analysis. This takes time.
- `_build/test/lib/$YOUR_PROJECT_NAME/ebin`: The directory in which the compiled
  BEAM files created by Mix are. Replace `$YOUR_PROJECT_NAME` with whatever your
  folder name happens to be.

Obviously, you could pass any options supported by Dialyzer here. Run `dialyzer
--help` to see them all. And that's that! You can now ensure that both you and
all your collaborators pay attention to Dialyzer.

## Additional Resources

- [Dialyxir][dialyxir]: Adds a `mix dialyzer` task. Also adds a `mix
  dialyzer.plt` task which can be very helpful when generating local PLTs on
  your dev machine.

- [Dialyzer docs][dialyzer]

[contracts-gem]: /elixir/ruby/2015/04/01/contracts-gem.html
[dialyxir]: https://github.com/jeremyjh/dialyxir
[dialyzer]: http://www.erlang.org/doc/man/dialyzer.html
[plts]: https://github.com/danielberkompas/travis_elixir_plts
