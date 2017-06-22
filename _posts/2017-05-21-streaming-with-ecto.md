---
layout: post
title: Streaming Data with Ecto
date:   2017-05-21
description: "See how to efficiently stream large datasets from a database using Ecto's Repo.stream."
tags:
  - elixir
  - ecto
---

**Note**: if you are not familiar with Elixir streams, you should [read the documentation](http://elixir-lang.org/getting-started/enumerables-and-streams.html#streams) on streams first or this post may be hard to follow.

It's a given that in your programming career, you will need to fetch a dataset and run some set of operations on it. With small datasets, you can often get away with fetching the records at once and doing what you need to do. But if you're trying to fetch a large dataset, you will run into some common problems. It's **very** slow, creates a blocking database connection, and most database libraries will load the dataset into memory which can slow down or kill your server.

Fortunately, Ecto [has a fantastic tool](https://hexdocs.pm/ecto/Ecto.Repo.html#c:stream/2) for handling this situation, [`Repo.stream/2`](https://hexdocs.pm/ecto/Ecto.Repo.html#c:stream/2). `Repo.stream/2` avoids fetching everything at once and instead fetches data in iterative cycles, performing operations on each record along the way.

In this example, we will work through a typical app feature: exporting database records as CSV files. It's a useful feature and is a good example to show how versatile Elixir streams can be.

First, let's look at some code:

{% highlight elixir %}

defmodule UserExporter do
  @columns ~w( id email inserted_at updated_at )a

  def export(query) do
    path = "/tmp/users.csv"

    Repo.transaction fn ->
      query
      |> Repo.stream
      |> Stream.map(&parse_line/1)
      |> CSV.encode
      |> Enum.into(File.stream!(path, [:write, :utf8]))
    end
  end

  defp parse_line(user) do
    # order our data to match our column order
    Enum.map(@columns, &Map.get(user, &1))
  end
end

{% endhighlight %}

Here we have a module with one public function, `export/1`, that accepts an Ecto query and returns back a file stream.

At the top, we declare a `@columns` module attribute to define the user data to show in our file. We'll need to refer to this `@columns` list later to reorder our data for the CSV file.

Next is the meat of the exporter. The first thing we do is open up a database transaction. This is a good idea when iterating over a large dataset and also a requirement when using `Repo.stream/2` with either a MySQL or Postgres. All work using `Repo.stream/2` needs to take place inside a transaction.

Having opened a transaction, we pass our query to `Repo.stream/2` which creates the initial stream. Remember, this only creates the stream; streams are lazily evaluated, so we haven't starting making any calls to the database yet.

We then pass the stream to a `Stream.map` function which will fetch and transform data from each user struct into a list of user data matching the order of our columns.

Using the wonderful [CSV library](https://github.com/beatrichartz/csv), we then pass the stream to the `CSV.encode/1` function to do all the hard work. `CSV.encode/1` both accepts and return streams, so it fits right into our pipeline setup here.

Finally, a call to `Enum.into` runs the whole stream, and the actual database calls start. By default, `Repo.stream/2` pulls down 500 records at a time, and each is parsed and written to a file at `tmp/users.csv`. Lastly, our function returns a file stream that points to our newly created file.  We can use this new stream to do more file-based operations or start some other stream-based work, like uploading the file to S3, etc.

As a test, we can see the results of our CSV in an iex console:

{% highlight elixir %}

iex> {:ok, file} = UserExporter.export(from u in User)
{:ok,
 %File.Stream{line_or_bytes: :line,
  modes: [:write, {:encoding, :utf8}, :binary], path: "/tmp/users.csv",
  raw: false}}
iex> File.read!(file.path)
#=> "1,joe@example.com,2017-04-27 21:57:42.972524,2017-05-20 18:48:16.235083\r\n2,jane@example.com,2017-04-27 18:36:32.053556,2017-04-27 18:36:32.065434\r\n3,jill@example.com,2017-04-27 18:37:43.503567,2017-04-27 18:37:43.503575\r\n"

{% endhighlight %}

Pretty neat, eh? Using streams, we created a simple and ordered pipeline to efficiently fetch data, transform it, and write it into a file - 500 records at a time. `Repo.stream/2` is a very handy tool dealing with large datasets, I highly recommend you give it a try if you haven't heard of it before.

