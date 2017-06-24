---
layout: post
title:  "Day 9: Using the Elixir Registry to perform locking of resources"
date:   2017-06-22
published: true
categories:
  - Elixir
  - Registry
  - Worker Pool
  - Locking
---

[In yesterday's blog post](http://blog.12startupsin12months.in/2017/06/22/day-8-the-beautiful-gen_stage/),
I shared how I got a worker pool working with a Queue and GenStage.
However, there was still one issue with that implementation. When there was more than one
(worker which is the case in my setup), it had the potential to run multiple user syncs
in multiple workers. Let us look at an example of how this happened.

  1. A sync event for user #1 arrives in the queue and is dispatched to worker#1.
  2. Another sync event for user #1 arrives and since the queue is empty, it gets filled up with this user id #1.
  3. And as soon as the queue has some data it is dispatched to the currently available workers, if we have 2 workers in our setup, this is dispatched to worker #2.

At this point we have 2 workers trying to sync the same user which is a recipe for disaster. One way to work around this is to use database locks provided by postgresql.
However, I didn't want to go down this route if it was possible.

## ETS to store current workers
At this point I contemplated using a simple ETS table to store the `{user_id, pid}` when a worker starts and removing it when a worker ends.
This would work. However, if a worker crashed after storing a `user_id`, it would permanently stop the `user_id`'s syncs from being processed.
I would then have to monitor the workers and clean up stuff if something crashed. However, at this point I was leaning more towards using the `Registry` as
it did clean up the data associated with a process when it crashed.

## Using the Registry
Using the Registry turned out to be much simpler than I thought.

I had to setup a supervisor for it in my `Application`

```elixir
  supervisor(Registry, [:unique, Danny.SyncQueue.Worker.registry_name]), # create registry to keep track of current worker jobs
```


### Worker code before using Registry for locking

```elixir

  def handle_events([uid], _from, state) do
     debug "syncing #{uid} on #{inspect self()}"
     UserSync.sync(uid)
     {:noreply, [], state}
  end

```

### Worker code after using Registry for locking

```elixir

  @registry_name :worker_wip
  def registry_name, do: @registry_name

  def handle_events([uid], _from, state) do
    get_lock_or_re_nq(uid, fn ->
      debug "syncing #{uid} on #{inspect self()}"
      UserSync.sync(uid)
    end)
    {:noreply, [], state}
  end

  defp get_lock_or_re_nq(uid, fun) do
    # register ourselves under the uid key
    case Registry.register(@registry_name, uid, :ok) do
      {:ok, _} ->
        # this means, we are good to go ahead and do our processing
        fun.()
        Registry.unregister(@registry_name, uid) # unregister once our function is done
      {:error, {:already_registered, pid} } ->
        # someone else is already working with this uid
        # let us re enqueue it so that it can be processed later
        debug("re nqing on account of another worker: #{inspect pid} processing this job. self: #{inspect self()}")
        spawn(fn ->
          :timer.sleep(:timer.seconds(1)) # sleep for a second before nqing, to avoid being quickly picked up by another worker
          Queue.nq([uid])
        end)
    end
  end

```

The process, is pretty simple. I let the workers pick up whatever user_ids are available in the queue.
However, when they start processing, I try to register this user_id in the Registry with the current process's pid.
If this user_id has already been registered with a different process, the Registry returns an `{:error, {:alreday_registered, pid } }`.
At which point, I wait a for a second and re enqueue it. I do the waiting in a different process to avoid blocking a worker.

I am close to the half month mark and am hopefully done with the difficult product stuff. I'll be spending more time trying to get feedback after releasing a private beta. Wish me luck!
