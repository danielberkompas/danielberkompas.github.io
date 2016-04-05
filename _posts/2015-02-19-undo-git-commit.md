---
layout: post
title: Undo a Commit in Git
category: git
date: 2015-02-19 07:00:00 -0800
---

I keep forgetting how to do this, so I thought I'd post this for my own future benefit:

```bash
// Completely deletes the most recent commit
$ git reset --hard HEAD~1

// Removes the most recent commit, but leaves changes intact. Useful if you
// might want to make a new commit.
$ git reset --soft HEAD~1
```
