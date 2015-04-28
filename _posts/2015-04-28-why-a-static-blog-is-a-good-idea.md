---
layout: post
title: "Why a Static Blog is a Good Idea"
---

Rather than use a popular blogging solution like [Wordpress][wordpress],
[Blogger][blogger], or even the new kid on the block, [Ghost][ghost], I chose
to use an old school method to write my blog: plain text.

This blog is a static site. My posts are written in plain text in a text editor 
on my computer, and are converted into simple HTML pages when I deploy. There is
no admin panel. There is no pretty online editor. Just text.

Why do I do this? Wouldn't it be better to use something less _manual_? There
are several reasons, and I think they're worth considering for anyone who wants
to write a blog.

## 1. Simplicity

There's nothing magical about this blog. It's just a collection of plain text
files which get converted into HTML files. There's nothing complex about a bunch
of plain text files in a folder, and so in that sense, it's easy to use.

Further, writing in plain text helps you focus. There are no shiny doodads to
distract you, no fonts to adjust, no colors to select. Because there's nothing
to do but write, you can find out quickly whether you're more interested in
having a pretty website or in _actually writing_.

## 2. Redundancy

Normally, your posts get subsumed into whatever blogging service you're using. 
If that tool dies for whatever reason, you could lose your posts. In contrast, 
my posts are stored locally on my computer, in my backup systems, and 
[on Github][github].  Because of this redundancy, there's much less chance I'll
lose anything important that I have written.

## 3. Portability

Since my posts are in plain text in a folder, they are _very_ portable. If I
decide to use a different system in 2030, it will be much less difficult to get
my posts out of the old and into the new system than it would be if I were using
Wordpress.

## 4. Speed

Since my blog ends up being a simple HTML site, it is very fast. The server I
host it on has to do very little work, which means that even a very modest host
can handle a lot of traffic.

## 5. Security

Since there is no admin portal, there are far fewer ways that my blog could be
hacked. There are no zero-day vulnerabilities in an HTML webpage. There are no
security patches to install.

Further, since I host this on my own domain, there is no need to get an SSL
certificate for the blog. I would need one if I were self-hosting a tool that
had an admin portal.

An attacker _could_ attempt to bring my blog down by making millions of requests 
against it. However, this is easily mitigated, because the entire blog can be 
moved to a global content delivery network _designed_ to serve millions of 
requests. What's more, this can be done cheaply and quickly.

## 6. Change Tracking

My blog is tracked with [Git](http://git-scm.org), a popular change-tracking
tool used by programmers everywhere. This means that _every_ change I make to my
blog posts is tracked. This revision history can be publicly available, 
[like mine is now][github-commits], or private, depending on your taste.

Personally, I think the world might be a better place if more bloggers opened
themselves up to this kind of accountability.

## 7. Few Compromises

I get all of the above with few compromises. I can still do all of the
following quite easily:

- Upload images
- Change the theme
- Reformat text
- Add dynamic behavior (like comments)

Granted, it's easier for me because web programming is my profession, but with
some help, most semi-technical people could do the same.

## Conclusion

So, that's why I think it's a good idea to have a static blog rather than use
Wordpress or some other tool. If you need pointers on how to get one set up, hit
me up [on Twitter][twitter], or check out [Jekyll][jekyll].

[ghost]: https://ghost.org
[blogger]: http://blogger.com
[wordpress]: http://wordpress.com
[markdown]: http://daringfireball.net/projects/markdown/
[github]: http://github.com/danielberkompas/danielberkompas.github.io
[github-commits]: https://github.com/danielberkompas/danielberkompas.github.io/commits/master
[twitter]: https://twitter.com/dberkom
[jekyll]: https://jekyllrb.com
