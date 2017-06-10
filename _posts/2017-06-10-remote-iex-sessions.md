---
layout: post
title: Creating IEx shells on Remote Nodes
date: 2017-06-10
description: "Learn how to connect to a remote elixir or erlang node with IEx from the command line."
categories: elixir
---

**Note**: `--remsh` has some security issues you should be aware of when using it on your local machine. Please be sure to read the **security section** at the bottom of this article.

Recently I learned about a fun option in IEx called `--remsh` or "remote shell". It starts an IEx shell within the context of a running Elixir node and also connects to that node's cluster, allowing you to debug and reproduce issues within a running application! Also note that despite the name, `--remsh` can connect to both local and remote nodes.

`--remsh` works in combination with two options, `--name` and `--cookie`. The first option creates a named IEx shell to identify yourself to other running nodes, and the second option is [the security cookie](http://erlang.org/doc/reference_manual/distributed.html#id88372) needed to access the node (often found under your home directory at `~/.erlang.cookie`).

To create a remote shell, run IEx like this:

{% highlight shell %}
# when running Elixir app on a server called example.com...
> iex --name "joe@iex_test" --cookie secret_cookie --remsh "node@example.com"

iex(node@example.com)1>
{% endhighlight %}

And you're in! You can see your named shell by running `Node.list`:

{% highlight shell %}
iex(node@example.com)1> Node.list
[:joe@iex_test]
{% endhighlight %}

You could even wrap this up in a simple bash script to make it easier to use:

{% highlight bash %}
function reiex() {
 if [ $1 = "staging" ]; then
  iex --name "$(whoami)@reiex" --cookie secret_cookie --remsh "node@staging.example.com"
 elif [ $1 = "prod" ]; then
  iex --name "$(whoami)@reiex" --cookie secret_cookie --remsh "node@prod.example.com"
 else
  echo "Environment not found!"
 fi
}
{% endhighlight %}

This way you can connect to a running instance anytime by doing `reiex staging`, etc. Very cool!

#### Security Issues

`--remsh` has security implications you should be aware of before using it. Remember - Elixir / Erlang nodes have complete access to all other nodes in a given cluster. So when you connect from your laptop via `--remsh`, other nodes in that cluster **can now access and make RPC calls on your local workstation**! If you connect to a compromised node from your local machine, your private files (even private SSH keys) would be up for grabs.

If you plan to do any serious work with remsh, I recommend reading Alex Weber's [writeup](https://broot.ca/erlang-remsh-is-dangerous) which goes in further detail about this issue. Alex also offers us a more secure alternative: create an SSH connection to your remote machine first and `remsh` locally to the node from there. This can at least prevent node RPC calls from being made against your workstation as well as establish a more secure connection between your machine and the node.

Happy `--remsh`-ing everyone!
