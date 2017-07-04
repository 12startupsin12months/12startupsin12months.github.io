---
layout: post
title: "Day 15: Refactoring, Technical Debt and Automated Tests"
date: 2017-07-04 17:48:36
comments: true
canonical_url: "..."
tags:
categories:
  - Refactoring
  - Technical Debt
  - Automated Tests
---

Today, was spent on refactoring my code and removing a lot of duplication and technical debt
and writing a shit load of automated tests. Elixir and Phoenix make writing tests very
straight forward. There isn't a lot of stuff to setup.

## Lessons, learnt
  - Write lots of automated tests, and have a CI run them. I did a post about
    [setting up Elixir with circleci a while ago](http://blog.12startupsin12months.in/2017/06/16/how-to-setup-ci-for-elixir-phoenix-using-circleci/)
    This has been such a life saver.
  - Don't worry too much about duplication early on in the project. This is something
    which [Sandi Metz talks a lot about](https://www.sandimetz.com/blog/2016/1/20/the-wrong-abstraction).
    If you come up with the wrong abstraction early on. You'll only dig yourself deeper into the mess.
  - Use tools to make your job easier. Elixir and Erlang have the awesome [dialyzer](https://github.com/jeremyjh/dialyxir)
    which analyzes your code and catches a lot of your bugs early on.

## Useful links
  - I started a blog called http://elixir.goodcode.in to share the interesting tidbits of elixir stuff.
    It already has a useful post: [How to run a chunk of code when your Elixir/Phoenix app starts](http://elixir.goodcode.in/2017/07/03/how-to-run-a-chunk-of-code-when-your-elixir-phoenix-app-starts/)
  - Dailyzer: https://github.com/jeremyjh/dialyxir
  - Sandi Metz Blog Post about [The Wrong Abstraction](https://www.sandimetz.com/blog/2016/1/20/the-wrong-abstraction)

## Questions Encountered
  - How to run stuff at startup the proper way in Elixir?
    *Blogged about it above.*
  - How to run dialyzer automatically in a CI on every push?
    *Haven't figured this one out yet.*

- - -
P.S: I'd love to hear your feedback on [sprymesh which is a simple web host (backed by Dropbox Synchronization) that I am building!](https://sprymesh.com)
