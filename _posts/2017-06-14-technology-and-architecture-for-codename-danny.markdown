---
layout: post
title:  "Day 3: Technology and Architecture for codename danny"
date:   2017-06-15 22:49:00 +0530
categories:
  - technology
  - stack
  - danny
  - architecture
---

Alright, today is the second day of my first startup. I have codenamed the project "danny", named after my second born :)

So, danny is going to be a web app which allows a typical user to host their websites using Dropbox. Let me first, lay down the objectives that danny is trying to achieve:

  1. A user should be able to quickly signup using their Dropbox account.
  2. Once they signup, a folder called "danny" should be added to their Dropbox "Apps" folder.
  3. Whenever they create a new site from danny's web console, a new folder should be created in `~/Dropbox/Apps/danny` with the name of the site. And, it should host this website on `http://[name-of-website].danny.com`.
  4. Once the website is setup. Any changes to the site's files on the user's computer should be deployed to a draft website. For instance `http://draft.[name-of-website].danny.com`.
  5. On the click of a button called "Publish", this draft website should be published to `http://[name-of-website].danny.com`

So, these are the minimum set of objectives that I have setup for danny. And, achieving this is going to require 3 large subsystems.

  1. The web console which allows users to authenticate using Dropbox and provides a UI.
  2. The *listener* sub system which listens to events from Dropbox when files change and keeps the data in sync.
  3. The *webhost* sub system which actually hosts the websites.

I am a huge fan of *[Elixir](https://elixir-lang.org/)* and *[Phoenix](http://www.phoenixframework.org/)*. And, Elixir lends itself very nicely to this particular problem.
One of the difficult parts of the app will be to listen for lots of incoming events and to download lots of files. Both of which are a piece of cake for Elixir.

Here is a quick diagram I drew which shows the communication between different parts of the system.

![danny diagram](/assets/danny-diagram-optimized.jpg)

All of these subsystems will be part of a single monolothic app initially. I may separate these towards the end.

So, currently I am thinking of a single monolothic Elixir/Phoenix app which stores data in a Postgresql database. And, a bit of javascript and css.
I have already started building out an app, and should have a working prototype within a week. However, writing the software seems to be the easy part :)
I also need to spend some time interacting with potential customers for feedback. That is going to be a teeny bit difficult. However, nothing in life is easy.

*Onward we go!*

