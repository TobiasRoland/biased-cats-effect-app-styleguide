# Battle-tested patterns for Scala with Cats Effects

A battle-tested, opinionated Styleguide for effective, safe, scalable Scala Cats Effect applications, prioritizing
legibility and
being welcoming to developers of different backgrounds and experience levels as your app(s) and microservice mesh grows.

## Why

Scala is a flexible langauge that has seen a lot of progress in the last few years, both in the core langugage and in
the Cats Effect/Typelevel ecosystem.
Not enough space is being used to document best practices for writing _applications_ rather than _libraries_ in Scala,
and this **is** a problem, because it makes it hard for newcomers to learn what non-library code looks like, and it
makes it easy for
people to get a wrong perception about the actual difficulty curve of Cats Effect and Typelevel libraries, which I will
try and
explain in terms of applied examples.

This repo is my attempt to document the best practices I've encountered in my work, and I hope that it will help someone
who
may be sceptical about why Scala is (IMO) the best current language to write microservice applications in, fall in love
with Scala and Cats Effect.

⚠️ This is a work in progress ⚠️

> This is ONE WAY to write legible services; others for sure exist! What I advocate here is a **solid start**;
> no rules are without exceptions. I encourage you to try it out on your next project - you can then make an active
> choice
> to break away from something. Ultimately,
> I don't believe programming should be a matter of dogma, but one of craftsmanship, and craftmanship
> is a skill acquired through deliberate practice and personal subjective appreciation and understanding of tradeoffs
> made. No perfect line of code exist that we can all agree on.

## Where is this advice coming from?

I think my perspective is one not talked about much in the Scala community; we tend to be mostly
focused around open source libraries, theoretical underpinnings of FP, rather than practical examples of how to
write applications. This is interesting, and I love this kind of content. However, I am not an open source maintainer,
and I am not a language contributor. What I am, though,
is a developer who has progressed from mid level, through senior, and into a tech lead role within the
Scala space. At this p I have worked on a **lot** of Scala applications, with a **lot** of scala developers. I'm familiar with
working with
services that count interactions in millions and billions, in a highly concurrent environment, scaling, ETL, Kafka,
redis, database tech... a lot of stuff, all centered around the connective tissue of Cats Effect and Scala.
I have upgraded services from Scala 2.11 to 3.x, and kept up with the typelevel stack's evolution, along the way. I've
been part of (and overseen) migrations, upgrades, tech-debt, legacy-systems and green-field development.

As for my credentials; I have my fingerprints on over two-hundred Scala microservices based around Cats Effect within
ITV, where I am currently employed as a tech lead for our Content and Personalisation teams. I say this not as a "I am
an infallable expert"
but rather as a "I have seen, written and discussed code with a series of excellent people around me, and I would like
to pay some of my learnings forward".

## Core philisophy

I am obsessed with reducing cognitive load for myself and for readers of my code, and I think
it's what makes for a long-lived service.

My core belief is that code first and foremost is literature - it's a piece of text that you
can read and understand, it should be written with purpose, and with clarity.
That doesn't mean excessive documentation, but it means constantly trying
to reduce the amount of information you need to know to understand the code, and to make it as easy as possible for
people
encountering your code to go "oh, of course that's what it does".

## Can I contribute?

You can certainly raise an issue if you have a question or a suggestion for improvement - however, I am unlikely to
accept pull requests changing the nature of the advice unless I'm personally convinced,
since this repo is (at this moment in time) meant to be a reflection of **my** personal opinion,
and coding patterns that I find appealing. I encourage you to fork this repo, or write your own styleguide, and share it
with the world. We need more app developers to learn from each other. I hope that this repo will help someone learn to
love Scala!


