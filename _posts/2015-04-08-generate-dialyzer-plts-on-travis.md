---
layout: post
title: "Build Dialyzer PLTs on Travis CI"
categories:
  - elixir
---

In a [previous post][previous], I wrote about how to easily get a prebuilt PLT
for your Elixir builds on Travis. But what if that doesn't work for you? What if
you have special requirements that my [prebuilt PLTs][prebuilt] don't meet?

## TL,DR;
You can fork my [Travis PLT generator][generator]. You'll just need an Amazon S3
bucket to store the resulting PLT files.

## How It Works
Since I'm not an expert at setting up Chef infrastructure, I knew that I
wouldn't be able to reliably get an identical virtual machine like the ones used
by Travis in production. So, building the PLTs locally was not an option.

It seemed both easier and more reliable to have Travis build the PLTs itself.
Since Travis is free for open source, I whipped up a simple repository and added
a `.travis.yml` that:

1. Runs `mix dialyzer` from the [dialyxir][dialyxir] package.
2. Uses the Travis [Artifacts][artifacts] script to upload the resulting file(s)
   to Amazon S3.

Since Travis allows running builds against multiple versions of Elixir and OTP,
getting a PLT for each possible combination is a cinch:

```yaml
language: elixir
elixir:
  - 1.0.0
  - 1.0.1
  - 1.0.2
  - 1.0.3
  - 1.0.4
otp_release:
  - 17.4
  - 17.3
  - 17.1
  - 17.0
```

You'll find more details in the [README][generator].

[dialyxir]: https://github.com/jeremyjh/dialyxir
[artifacts]: https://github.com/travis-ci/artifacts
[previous]: http://blog.danielberkompas.com/elixir/2015/04/08/generate-dialyzer-plts-on-travis.html
[prebuilt]: https://github.com/danielberkompas/travis_elixir_plts
[generator]: https://github.com/danielberkompas/travis_elixir_plt_generator
