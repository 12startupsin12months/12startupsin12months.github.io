---
layout: post
title:  "Day 5: First working prototype of the webserver"
date:   2017-06-19
published: true
comments: true
categories:
  - Elixir
  - Retrospective
  - Webserver
  - Cowboy
  - Cache
  - danny
---

Erlang/Elixir make it too easy to build industry grade webservers. There are so many pieces of erlang code that
make writing robust and highly performant code a piece of cake.

Today, I wrote a simple resource webserver which serves the web pages for danny. The objectives of this
webserver were:

  1. Use caching to do very quick url lookups and fallback to the DB if the cache is not filled yet.
  2. Be performant while serving web pages.

## 1. Caching

This was very easy to implement, thanks to the awesome `:ets` from erlang.

The following block of code forms most of the cache module:

```elixir

defmodule Cache do
  @resource_tab :resource_cache

  # client api
  def filepath_for(subdomain, path) do
    # lookup in cache
    case cached_filepath_for(subdomain, path) do
      {:ok, _, _} = found -> found
      :not_found ->
        # lookup in database
        case Site.find_by_subdomain(subdomain) do
          nil -> :not_found
          site ->
            debug("DB_HIT #{inspect url(subdomain, path)}")
            cache_entries(site)
            cached_filepath_for(subdomain, path)
        end
    end
  end

  defp cached_filepath_for(subdomain, path) do
    case :ets.lookup(@resource_tab, url(subdomain, path)) do
      [{_, content_hash}] ->
        debug "CACHE_HIT #{inspect {subdomain, path}}"
        {:ok, content_hash, ResourceStore.Store.path_for_content_hash(content_hash)}
      [] ->
        debug "CACHE_MISS #{inspect {subdomain, path}}"
        :not_found
    end
  end

  # ....

  use GenServer
  def start_link do
    GenServer.start_link(__MODULE__, [])
  end

  def init(state) do
    :ets.new(@resource_tab, [:named_table, {:write_concurrency, true}, {:read_concurrency, true}, :public])
    {:ok, state}
  end

  # ...

end

```

## 2. Serving pages performantly
This too was very straight forward thanks to the awesome cowboy server.
My phoenix app now has a second endpoint which is a plain plug and runs on a separate port from the main app.

```elixir
defmodule Danny.ResourceServer.Plug do
  import Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    subdomain = subdomain conn
    path = conn.request_path
    case Cache.filepath_for(subdomain, path) do
      :not_found ->
        conn
        |> put_resp_content_type("text/html")
        |> send_resp(404, "<!doctype><h1>404 Not found</h1>")
      {:ok, content_hash, content_path} ->
        conn
        |> put_resp_content_type(content_type_for_path(path))
        |> put_resp_header("etag", content_hash)
        |> send_file(200, content_path)
    end
  end

  # ...
end
```

This code uses `send_file` which should circumvent the file's bytes from even being read into the BEAM. Elixir never ceases to be fun. These two modules form the bulk of the webserver. Some quick benchmarking using wrk showed good results. However, putting this on a production server and benchmarking it properly is something I'll do in the future.

This week should see the deployment of our initial prototype :) I am very excited about this :) I'd love to hear your feedback once a usable version is online :)

*P.S I'd love to hear your thoughts on the features you would love to have in a web host which is synced using Dropbox*
