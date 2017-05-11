---
layout: post
title:  "Elixir Nested Modules and Auto-Aliasing"
date:   2017-05-10
description: "Nesting modules in Elixir has a few gotchas you should be aware of."
categories: elixir
---

---
layout: post
title:  "Elixir Nested Modules and Auto-Aliasing"
date:   2017-05-10
description: "Nesting modules in Elixir has a few gotchas you should be aware of."
categories: elixir
---

Recently, I was coding a CSV exports module when I came across an unexpected error in Elixir. Using the [CSV](https://github.com/beatrichartz/csv) library, I wrote something like this:

{% highlight elixir %}

# mix.exs
defp deps do
  [
    {:csv, "~> 1.4.2"}
    ...
  ]
end

# csv.ex
defmodule Foo do
  defmodule CSV do
    def export(file) do
      CSV.encode(file)
    end
  end
end
{% endhighlight %}

I expected `Foo.CSV.export(file)` to trigger the `CSV` library to encode the file I passed into the function.  To my surprise, the app came back with this error:

{% highlight elixir %}
(UndefinedFunctionError) function Foo.CSV.encode/1 is undefined 
or private
{% endhighlight %}

I realized this error makes some sense. I'm calling a function on a `CSV` module, and I *just* defined a named `CSV` module right above it. Naturally, the app should be confused as to which `CSV` module I'm referring to. But I really didn't want to change my module's name, so how can I tell the app which module I want to use?

After some experimenting, I found something even more interesting - switching from a nested module to a single namespaced module made it work:

{% highlight elixir %}

# csv.ex
defmodule Foo.CSV do
  def export(file) do
    CSV.encode(file)
  end
end

{% endhighlight %}

What?? I thought nested modules and namespaced modules compiled to the same thing? Shouldn't the first example also compile to `Foo.CSV`? And why does the second example not throw the same error as the first one?

### Why this happens

After asking around, I found this behavior puzzled other Elixir developers as well. Thankfully, [Bryan Joseph](http://github.com/bryanjos) helped me figure out what was happening.

While compiling, when Elixir reaches a nested module, it creates an "auto-alias" for that nested module.  In other words, this code:

{% highlight elixir %}
defmodule Foo do
  defmodule CSV do
    def export(file) do
      CSV.encode(file)
    end
  end
end
{% endhighlight %}

actually compiles to something more like this behind the scenes:

{% highlight elixir %}
defmodule Elixir.Foo do
  defmodule Elixir.Foo.CSV do
    def export(file) do
      CSV.encode(file)
    end
  end
 
  alias Elixir.Foo.CSV, as: CSV
end
{% endhighlight %}

This is why my original example threw an error - when `CSV.encode(file)` is called, the auto-alias directs the app to instead look for an `encode/1` function on the `Elixir.Foo.CSV` module.

Yet, when modules are not nested, the compiler does not create an auto-alias, so there is no confusion about which `CSV` module I'm trying to reference.

So if you are hitting a similar error with nested modules, consider either changing the name or switching over to a namespaced module instead.