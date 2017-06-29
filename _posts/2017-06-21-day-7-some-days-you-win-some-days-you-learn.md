---
layout: post
title:  "Day 7: Some days you win, Some days you learn"
date:   2017-06-21
published: true
comments: true
categories:
  - Elixir
  - GenStage
  - Data Race
---

The Dropbox API is mostly nice to work with. It notifies integrations of file changes using web hooks.
And, Dropbox needs the webhook to respond immediately, which means the post should be processed in some kind of a background job.
Fortunately, Elixir makes it trivial to run a background job. You just need to spawn a process to run something in the background.

```elixir
defmodule WebhooksController do
  # ....
  def sync(conn, params) do
    spawn(fn -> process(params) end)
    text(conn, "ok")
  end
  # ....
end
```

It is that simple. However, if a user makes changes too quickly you end up receiving multiple POSTs from Dropbox which is to be expected.
Now, it is your responsibility to make sure that you don't have 2 separate processes syncing the same user. That is a recipe for data races.
Elixir has an awesome abstraction just for this kind of use case, "The GenStage". However, setting up is not trivial. I waddled through a lot of examples
to try and understand how to set it up for my use case. I haven't got it working just right yet. Hopefully I'll have it working by tomorrow.

**Some days you win, some days you learn** - Some Wise Guy.
