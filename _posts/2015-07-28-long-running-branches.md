---
layout: "post"
author: "Daniel Berkompas"
title: "Avoid Long-Lived Feature Branches"
categories:
  - scm
date: 2015-07-28 07:00:00 -0800
---

Today, I intend to rail against the evils of long-lived feature branches. Having collaborated on a number of projects where they happened, I'm now convinced that they are almost always the wrong way to go.

<!-- more -->

## Why We Do Long-Lived Branches

The reasoning behind these branches is simple. Your boss wants the program to keep running like it is, but he wants a large, complex feature to be implemented that will touch a lot of the codebase.

Many teams respond to this by creating a feature branch in source control, in which the feature will be developed until such a time that it is "ready". Normal development continues in the main branches while the feature is being developed.

## What's Wrong With Them

There are many problems with this approach, but I think they can be summarized as two large problems.

First, **code review becomes messy and ineffective.** When the feature branch is finally merged into the `master` branch, the diff is too large to review effectively. I've been handed diffs like this to review before, and my eyes glaze over after about 300 lines of code. The sheer volume of code to review reduces the quality of my review, because I don't want to sit and review code all day, so I rush through it. Rushed code reviews are worse than no code review at all, because they give the illusion that code has been reviewed when in fact it has not.

Second, **merging becomes hard.** The longer a branch lives, the more different the main branch becomes, and the more difficult it can be to keep the two in sync. You can mitigate this by doing frequent merges and rebases in from `master`, but many developers neglect this.

In the end, you have a giant blob of code that isn't very well tested or reviewed, and which probably conflicts with your `master` branch in a number of significant ways. In the projects I worked on, this either resulted in serious bugs or significant delays getting the feature done.

## A Better Way

Like many others have written, I recommend that you develop each feature as incrementally as possible, merging as often as possible into your `master` branch. Ensure the tests are all passing after each incremental change. Put the feature behind a feature flipper if possible, or, if the feature changes too many things, start a new "main" branch where all development will occur and use `master` only for hotfixes until the feature is done.

This approach ensures that pull requests are small, and therefore makes code review a meaningful process. Also, there's no fear that `master` will get very far ahead in the two or three hours that each little feature branch lives.

So, please, no more massive pull requests! Keep your branches small, your
commits focused, and merge often.
