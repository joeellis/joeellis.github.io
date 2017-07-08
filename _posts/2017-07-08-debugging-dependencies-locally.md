---
layout: post
title: Debugging Dependencies Locally
date: 2017-07-08
description: "Elixir and Mix makes it dead simple to debug dependencies."
tags: elixir
---

Have you ever needed to debug a dependency, perhaps to track down a bug or maybe just to see how it works? In scripted languages like Ruby or JS, you can usually add debug statements, like `puts "test"` or `console.log("test")`, to the dependency's code and see the output the next time you run your app.

This does not quite work in an Elixir / Mix project as dependencies are compiled once. Fortunately for us, [`mix`](https://hexdocs.pm/mix/Mix.html (Elixir's build tool) gives us an option called `path`, to automatically recompile a dependency whenever it changes. Simply declare the `path` option with your dependency in your project's `mix.exs` like so:

{% highlight elixir %}
defp deps do
  [
    ...
    {:my_dep, '~> 1.0.0', path: 'deps/my_dep'}
    ...
  ]
end
{% endhighlight %}

This tells `mix` to look at the `deps/my_dep` folder for that dependency when compiling, and recompile it (and related project files) anytime something in there changes. Now you can open up the code at `deps/my_dep`, add your debug statements, and explore away!

### Absolute & Relative Paths

The `path` option isn't limited to looking at the `deps` folder, and will also accepts relative and absolute system paths, like so:

{% highlight elixir %}
defp deps do
  [
    ...
    {:my_dep, '~> 1.0.0', path: '../path/to/my/local/my_dep'}
    ...
  ]
end
{% endhighlight %}

This is particularly useful if you are developing a Hex package along side your project, or if you want to clone the package locally to track your changes and submit patches later.

### Reset the Dependency

Once you're finished debugging and want to restore the dependency to it's normal state, simply clean it out and re-fetch it by running:

{% highlight elixir %}
mix do deps.clean my_dep, deps.get, compile
{% endhighlight %}

And everything will be reset back to how it was in the beginning.
