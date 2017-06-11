---
layout: post
title: Connect to Running Elixir Applications with IEx Remote Shell
date: 2017-06-10
description: "Using the remsh option in IEX to connect to Elixir applications from the command line."
categories: elixir
---

**Note**: `--remsh` has security issues you should be aware of when using it on your local machine. Please be sure to read the **security section** at the bottom of this article.

Recently I learned about a fun option built into IEx called `--remsh` or "remote shell". It creates an IEx shell in the context of an Elixir node, allowing you to debug and reproduce issues inside a running application! Also note that despite the name, `--remsh` can connect IEx to either local or remote nodes / applications. Here's an example of how it works:

{% highlight shell %}
# when running Elixir app on a server called example.com...
> iex --name "joe@iex_test" --cookie secret_cookie --remsh "node@example.com"

iex(node@example.com)1> Node.list
[:joe@iex_test]
{% endhighlight %}

`--remsh` works in combination with two other options, `--name` and `--cookie`. The first is required and creates a named IEx shell to identify yourself to other nodes in the cluster. The second is not required but usually needed - it's the [the security cookie](http://erlang.org/doc/reference_manual/distributed.html#id88372) needed to access the node (often found under your home directory at `~/.erlang.cookie`).

You could also wrap this up in a simple bash script to make it even easier to use:

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

Just run `reiex <environment name>` to connect to your staging or production servers.

#### Security Issues

`--remsh` has security implications you should be aware of before using it. Remember - Elixir / Erlang nodes have complete access to all other nodes in a given cluster. So when you connect from your laptop via `--remsh`, other nodes in that cluster **can now access and make RPC calls on your local workstation**. If you connect to a remote node that has been compromised from your local machine, your private files (even private SSH keys) would be up for grabs.

If you plan to do any serious work with remsh, I recommend reading [Alex Weber's writeup](https://broot.ca/erlang-remsh-is-dangerous) which goes in further detail about this issue. Alex also offers us a more secure alternative: create an SSH connection to your remote machine first and `remsh` locally to the node from there. This can at least prevent node RPC calls from being made against your workstation as well as establish a more secure connection between your machine and the node.

Happy `--remsh`-ing everyone!
