---
layout: post
title: Think About the Next Guy
category: best-practices
---

> I hated all my toil in which I toil under the sun, seeing that I must leave it
> to the man who will come after me, and who knows whether he will be wise or a
> fool? Yet he will be master of all for which I toiled and used my wisdom under
> the sun. - Ecclesiastes 2:18-19

Everyone dies, and the world is constantly changing. Therefore, if anything is
to be carried on into the future, there must be a _succession_. Someone new must
take up the mission, the art, or the craft, or it will die with the current
generation.

<!-- more -->

Programmers should be very familiar with this. How many formerly popular
frameworks have been left in the dust? How many popular applications have
disappeared? How often have you rewritten old codebases, not necessarily because
they were bad, but because they were undocumented and you didn't understand 
them? 

How often has a client come to you, wanting just one feature added to their
software, and you ended up telling them the whole thing had to be rewritten? How
often have you inherited code from previous developers? How often was that a
pleasant experience? How often has a key employee left your company, taking all
his knowledge with him, leaving new developers to pick up the pieces? How often
has an open source project you rely on been abandoned, forcing you to maintain
your own fork?

If your career has been anything like mine, I'm guessing that all these things
have happened to you, and they weren't fun. 

## We are really, really bad at succession

Most programmers give very little thought to succession. We bang out our code,
merrily implementing new features and starting new frameworks, rarely thinking
about the long-term implications of what we're doing.

As a result, our projects don't have READMEs, they have no high-level 
documentation, no low-level unit documentation, no instructions for new 
developers, and often few tests. Further, we write them in whatever is new and 
shiny, rarely thinking about whether the new and shiny will still be shiny as
little as six months from now.

If we have any documentation at all, it's usually directed toward our end-users,
and talks about how to _use_ the system. The system itself remains an 
undocumented black box.

## Why it matters

**Code that no one understands is crap.** It doesn't matter what language it is
written in, what patterns it uses, or how shiny it was at the time. It's crap.
It gets thrown out.

**Code that people understand and can extend is good.** Again, it doesn't matter
what it is written in, or whether it has curly braces and semi-colons or not.

**Rewrites are wasteful.** Resources that could be used building new
functionality are instead being used to reproduce old functionality. It is
therefore better to extend than rewrite.

**Rewrites produce worse code as often as not.** I know we always feel like 
we've made something better when we rewrite it, but this isn't always true. 
The guy whose code you're rewriting probably thought that he was improving
things too.

**Businesses can't afford endless rewrites.** Increasingly, programmers with a 
proven track record of writing maintainable software will get the jobs or the 
contracts, even if they're not using the new hotness. We get by now due to the
fact that programmers are in high demand and people are throwing lots of money
at technology to see what will stick. This won't last forever. Eventually, our
clientele will become more educated and ask harder questions.

**Eventually, you've got to settle down.** Programmers get old and die like
everyone else. Not all of us are going to be able to start our own business or
have a comfortable retirement. This means you probably face three options at
retirement age:

1. Get into management.
2. Keep chasing new trends when you're 60. The kids will be better than you 
   at this because they always are.
3. Make a way for yourself writing maintainable software that stays relevant 
   over the long term. Let the kids play with the new toys.

Taking any of these options other than #2 will require planning and preparation,
_before_ you hit retirement age.

## What you can do about it

There are some simple disciplines that I think will help keep your code
maintainable in the long run.

1. **Think about the next guy who's going to work on your code.** You might 
not know him. You might not be there for him. So, what does he need to know 
about your code so that he doesn't just throw it out and waste time rewriting 
it? Tell him. Make sure he knows how _to even get it running._

2. **Don't get dazzled so quickly by new technology.** New is not always
better. Keep a firm bias toward proven tools and let the new ones mature for a
few years. Try not to get locked into a technology that might be discontinued 
any time soon.

3. **Design your code for extensibility.** Your code will certainly need to
change, so try to foresee what types of changes will be required. For example,
if it's a reporting app, it's foreseeable that someone will want a new report.
Make it easy to add one, and then document how to do it. Isolate the areas of
the code that need to change often from the areas that don't.

I admit I haven't always practiced these disciplines, but the older I get, the 
more value I see in them. They can only make you a better developer, and ensure 
that those who come after you can succeed. We need a lot more programmers who 
think about the next guy.

Expect to see more posts on this topic.
