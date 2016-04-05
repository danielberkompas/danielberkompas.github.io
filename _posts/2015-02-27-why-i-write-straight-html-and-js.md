---
layout: post
title: Why I Write Straight HTML and JS
categories:
  - html
  - js
date: 2015-02-27 07:00:00 -0800
---

It's very popular in the Rails community to use templating languages that compile down to HTML. The asset pipeline likewise makes it very easy to use Coffeescript and SCSS, or other languages that compile down to Javascript and CSS.

When I got started with Rails, I was just as much a fan of these technologies as everyone else. Over time though, they've lost their shine, and I'll explain why.

<!-- more -->

## 1. Whitespace

Nearly all these languages achieve their "simplicity" through whitespace. They make code shorter by removing closing tags, whether those be `</p>` tags or javascript `}`. You end up with code nesting that looks something like this:

```slim
html
  head
    title My first HTML project
    meta content="..."
    link href="..." type="text/css"

  body
    p Hello World
```

At first glance, this looks wonderful. Unfortunately, most code ends up looking more like this over time:

```slim
.span2
  h1 class="#{icon_color_class}": i class="#{icon}"
.span10
  h5.pad-bottom-small= text
  ul.unstyled
    - if collection.size > 0
      - collection.each do |item|
        li= item.name
    - else
      li (None)

  = link_to link, url, class: link_color_class
```

Here are the problems I have with this:

- **Nesting is messy.** In HTML, it's very easy to format code however you want. Templating languages all have different funky ways to accomplish this.

- **The code is hard to scan.** Without end tags, it's much harder to tell what the structure of the file should be.

- **Growth makes things worse.** The larger your templates grow, the harder it is to see what is going on.

- **It's ugly.** These templating languages are advertised based on how pretty they are, but in practice, they can look much worse than straight HTML.

## 2. They're Distracting

These abstractions distract developers in the following ways:

- **Documentation**.  Most documentation and tutorials are written in the base languages of the web, and developers using Slim, HAML, Emblem, or Cofeescript then have to spend time figuring out how to do the same thing.  If they were just writing HTML and Javascript, they could just copy-paste and move on.

- **Debugging.** Debugging can become much more complicated, because what you see in your codebase is not actually what gets run in the browser. Bugs crop up due to the fact that your attempt doesn't compile into the same code as the documentation you were following. Getting help is harder, because other people on the internet aren't using the specific combination of technologies that you are, and therefore your specific issue is rare.

- **Context Switching.** The developer must keep Javascript and HTML in his mind as he writes the compiled language. This means that he has two languages in his head as he codes, rather than one. Arguably, this is harder on the brain than just writing straight Javascript and HTML.

## 3. Speed

If you write straight Javascript and HTML, your asset compilation times will be faster. In production, your views will render more quickly, as the server or browser must do much less work to generate HTML.

## 4. High Maintenance Costs

If you maintain a lot of projects, having lots of templating languages floating around can be a large burden. New developers have to get up to speed on the templating language (which slows them down) and it's harder to share code between codebases due to the dependencies that code brings with it.

## What About the Features?

Some of these languages offer features not available in their underlying
language. SCSS is a good example. This used to be true of CoffeeScript, but now that ES6 has adopted most of its good ideas, and transpilers are readily available, this is no longer a compelling reason to use CoffeeScript.

I still use SCSS because its features are so useful, and its learning curve is so low due to the fact that CSS is valid SCSS. In my experience, SCSS is the exception rather than the rule, however. If your language provides _actual features_ rather than syntactic sugar, I'd consider keeping it. Otherwise, I'd throw it out.

You already know Javascript. You already know HTML. So does everyone who will work on your project now or in the future. Why not speak the common language?
