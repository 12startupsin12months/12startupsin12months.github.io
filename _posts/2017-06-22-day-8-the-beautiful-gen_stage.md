---
layout: post
title:  "Day 8: The beautiful GenStage"
date:   2017-06-22
published: true
comments: true
categories:
  - Elixir
  - GenStage
  - Worker Pool
---

So, I finally grokked the GenStage abstraction! Yay!

GenStage has a lot of moving parts and is a bit difficult to understand because of all the ways in which it can be setup. My use case however, was very simple:

I wanted a worker pool and queue which would work in tandem. Like I discussed yesterday, Dropbox sends a webhook whenever a user changes their files.
And, these changes need to be then parsed and synced properly. In my use case, all I needed was:

  1. A Queue: Which would accumulate user ids posted to our webhook endpoint.
  2. N Workers: Where each worker would ask the queue to give a single user id, and then sync that user.

In GenStage terms, I had a single producer(Queue) and multiple consumers(Worker). This could be easily setup with the following code:

The accompanying code for this post is at: https://github.com/12s12m/gen_stage_demos/

```elixir
defmodule Q do
  use GenStage

  def start_link do
    # initialize our GenStage with a queue as its state
    GenStage.start_link(Q, :queue.new, name: Q)
  end

  def init(state) do
    # returning this specific tuple marks this process as a producer
    {:producer, state}
  end

  def handle_demand(demand, q) when demand > 0 do
    # dequeue as many jobs from our queue as the demand
    {jobs, {q, pending_demand}} = dq_jobs(q, demand, [])
    # return those jobs and our new queue
    {:noreply, jobs, q}
  end

  # this is one of our return paths, since we pop elements and prepend them
  # to our accumulator, the acc will have the elements in the reverse order in which they were popped
  # so, to fix the order of the popped jobs, we need to reverse them
  def dq_jobs(q, 0, acc), do: {Enum.reverse(acc), {q, 0}}
  def dq_jobs(q, n, acc) when n > 0 do
    case :queue.out(q) do
      # :queue.out returns an {:empty, q} when our queue is empty
      # in which case we need to reverse our jobs and return from this function
      {:empty, _} -> {Enum.reverse(acc), {q, n}}
      # however, if we do have elements, we can recursively call ourselves
      # after getting the first element from the queue
      { {:value, job}, q} -> dq_jobs(q, n-1, [job | acc])
    end
  end

end


defmodule Worker do
  use GenStage

  def start_link() do
    GenStage.start_link(Worker, :ok)
  end

  def init(:ok) do
    # the first atom `:consumer` marks this stage as a consumer
    # we are also doing the subscription in the init, so if this process
    # crashes and is restarted the subscription happens
    # we are also setting the max_demand to 1 because we can only work on one user sync at a time
    # by default GenStage will set max_demand to 1000 which means it will request a 1000 jobs in one go from the producer
    # which is definitely not what we want
    {:consumer, :ok, subscribe_to: [{Q, max_demand: 1}] }
  end

  def handle_events(events, _from, state) do
    IO.inspect({"processing", self(), events})

    # do some heavy lifting
    Process.sleep(1000)

    # consumers don't return events
    {:noreply, [], state}
  end
end

defmodule MyApplication do
  use Application

  def start(_type, _args) do
    import Supervisor.Spec, warn: false

    # Define workers and child supervisors to be supervised
    children = [
      worker(Q, []),
      worker(Worker, [], id: 1),
      worker(Worker, [], id: 2),
      worker(Worker, [], id: 3),
    ]

    # See http://elixir-lang.org/docs/stable/elixir/Supervisor.html
    # for other strategies and supported options
    opts = [strategy: :one_for_one, name: Gsq.Supervisor]
    Supervisor.start_link(children, opts)
  end
end

```

You can check this by cloning https://github.com/12s12m/gen_stage_demos and running `mix run lib/first.exs`

This setup is fine and dandy. However, our queue is initially empty. And, there is no way to fill it from outside at the moment.
So, we need to write some code which allows adding jobs to our Queue.

```elixir
defmodule Q do
# ...

  def nq(jobs) do
    GenStage.call(__MODULE__, {:nq, jobs})
  end

  def handle_call({:nq, jobs}, _from, q) do
    # nq new jobs
    q = Enum.reduce(jobs, q, fn job, q -> :queue.in(job, q) end)
    {:reply, :ok, [], q} # dispatch immediately
  end
# ...
end

defmodule MyApplication do
# ...

  def start do
    # nq a few jobs
    Q.nq(1..10 |> Enum.to_list)
  end
# ...
end

```

Our workers aren't processing any jobs even after we enqueue a few jobs.
And when I run, `mix run lib/second.exs` I see the following:

```
13:43:22.988 [debug] setting up the supervisor

13:43:22.988 [debug] starting q

13:43:22.990 [debug] starting worker

13:43:22.990 [debug] starting worker

13:43:22.992 [debug] handling demand: 1 q: {[], []}

13:43:22.992 [debug] handling demand: 1 q: {[], []}

13:43:22.993 [debug] nqing jobs: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10], {[], []}

```

Which shows that the our nq function is being called is happening. Let us add a debug method to check what the value of our queue is after nqing.


```elixir
defmodule Q do
# ...
  def peek do
    GenStage.cast(__MODULE__, :peek)
  end

  def handle_cast(:peek, state) do
    debug "STATE: #{inspect state}"
    {:noreply, [], state}
  end
# ...
end
```

Now, if we run `mix lib/three.exs`. We see the following:

```
13:48:42.285 [debug] setting up the supervisor

13:48:42.286 [debug] starting q

13:48:42.287 [debug] starting worker

13:48:42.288 [debug] starting worker

13:48:42.297 [debug] handling demand: 1 q: {[], []}

13:48:42.297 [debug] handling demand: 1 q: {[], []}

13:48:42.298 [debug] nqing jobs: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10], {[], []}

13:48:42.298 [debug] STATE: {[10, 9, 8, 7, 6, 5, 4, 3, 2], [1]}
```

Which shows that our queue is properly setup.

This is where I was left scratching my head for a very long time. I was wondering why our workers weren't asking for more work now that our queue is full. That is when I stumbled upon this helpful issue https://github.com/elixir-lang/gen_stage/issues/80
where Jose talks about storing the demand and then responding to that when we have enough jobs. Go ahead and read that whole issue, it has some good information.

Now, with this new knowledge we need to tweak our code so that when a worker asks for something and we don't have it, we just increment a number in our state to keep track
of how many jobs we need to emit once we have enough jobs.

```elixir
defmodule Q do
# ...
  def start_link do
    # our state will now have to keep track of a pending demand
    GenStage.start_link(__MODULE__, {:queue.new, _pending_demand = 0}, name: __MODULE__)
  end

    # ...
  end

  def handle_call({:nq, jobs}, _from, {q, pending_demand}) do
    debug "nqing jobs: #{inspect jobs}, #{inspect q}"

    # nq new jobs
    q = Enum.reduce(jobs, q, fn job, q -> :queue.in(job, q) end)

    # and dispatch pending demand
    {jobs, {q, pending_demand}} = dq_jobs(q, demand, [])
    # instead of emitting [] as the jobs, we are now getting pending_demand number of jobs
    # and returning it

    {:reply, :ok, jobs, {q, pending_demand}}
  end

  def handle_demand(demand, {q, pending_demand}) when demand > 0 do
    debug "handling demand: #{demand} q: #{inspect q}"
    # dequeue as many jobs from our queue as the demand
    {jobs, {q, _pending_demand}} = dq_jobs(q, demand + pending_demand, [])
    # return those jobs and our new queue
    {:noreply, jobs, q}
  end
# ...
end



MyApplication.start()
spawn(fn ->
  Enum.each 1..10, fn idx ->
    # nq a few jobs
    jobs = (idx * 10)..((idx+1) * 10)
    Q.nq(jobs |> Enum.to_list)
    Q.peek
    Process.sleep(:timer.seconds 10)
  end
end)
Process.sleep(:infinity)
```

We have also added some code which enqueues new jobs every 10 seconds. Running `mix run lib/fourth.exs`

Whew! That was a real long post. I wanted to document how to use gen stage because it can be a bit intimidating because of its versatility.
