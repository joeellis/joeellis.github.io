---
layout: post
title:  "Using Circle 2.0 with Elixir and Phoenix"
date:   2017-09-12
description: "Making the most of CircleCI 2.0 to create an automated CI for Elixir tests"
tags:
  - elixir
  - phoenix
  - circleci
---

I recently took advantage of CircleCI's new 2.0 features in a Phoenix application and thought it was worth sharing here.  CircleCI 2.0 [boasts many new features](https://circleci.com/docs/2.0/#features) but the most interesting ones to me were the native support for Docker images and their advanced caching features.

It turns out to be a great move - both of these features helped cut our build times by at least *50%*. Using a Docker iamge meant CircleCI no longer needed to compile Elixir, Erlang, and Node for each job. And the advanced caching features went a step further by giving us control over how our `build` and `deps` directories were cached, saving us on the compilation time between jobs. It did take some research through their docs and forums to figure out how to create a complete, working CircleCI yaml file, so I wanted to write up what I did in case my example helps save time for other people.

To start, assume we have a brand new Phoenix app that uses the following:

- Elixir 1.4.2, Erlang 19.x, Node 7.x, and `yarn` compiled into a single Docker image
- PostgreSQL for the database

Your stack may differ from the above, and that's ok. The strategy outlined here should work for the majority of Phoenix projects. Just know there may be a few things you'll have to change to make this config work for your project.

To start, here is what the full CircleCI yaml config file looks like:

{% highlight yaml %}
{% raw %}
# .circleci/config.yml

version: 2
jobs:
  build:
    docker:
      - image: joeellis/elixir-phoenix-node:1.0
        environment:
          - MIX_ENV=test
      - image: postgres:9.6.2-alpine
        environment:
          - POSTGRES_USER=my-database-user
          - POSTGRES_PASSWORD=my-database-password
          - POSTGRES_HOST=localhost
    working_directory: ~/app
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
            - v1-mix-cache-{{ .Branch }}
            - v1-mix-cache
      - restore_cache:
          keys:
            - v1-build-cache-{{ .Branch }}
            - v1-build-cache
      - run: mix do deps.get, compile
      - save_cache:
          key: v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
          paths: "deps"
      - save_cache:
          key: v1-mix-cache-{{ .Branch }}
          paths: "deps"
      - save_cache:
          key: v1-mix-cache
          paths: "deps"
      - save_cache:
          key: v1-build-cache-{{ .Branch }}
          paths: "_build"
      - save_cache:
          key: v1-build-cache
          paths: "_build"
      - restore_cache:
          keys:
            - v1-yarn-cache-{{ .Branch }}-{{ checksum "assets/yarn.lock" }}
            - v1-yarn-cache-{{ .Branch }}
            - v1-yarn-cache
      - run:
          working_directory: assets
          command: yarn install && yarn test
      - save_cache:
          key: v1-yarn-cache-{{ .Branch }}-{{ checksum "assets/yarn.lock" }}
          paths: assets/node_modules
      - save_cache:
          key: v1-yarn-cache-{{ .Branch }}
          paths: assets/node_modules
      - save_cache:
          key: v1-yarn-cache
          paths: assets/node_modules
      - run: mix ecto.create && mix ecto.migrate
      - run: mix phoenix.digest
      - run: mix test
{% endraw %}
{% endhighlight %}

That is a bunch of config, so let's break it down piece by piece and see what's going on:

{% highlight yaml %}
{% raw %}
# .circleci/config.yml

version: 2
jobs:
  build:
    docker:
      - image: joeellis/elixir-phoenix-node:1.0
        environment:
          - MIX_ENV=test
      - image: postgres:9.6.2-alpine
        environment:
          - POSTGRES_USER=my-database-user
          - POSTGRES_PASSWORD=my-database-password
          - POSTGRES_HOST=localhost
    working_directory: ~/app
{% endraw %}
{% endhighlight %}

First we tell CircleCI that we'd like our build to execute under the [`joeellis/elixir-phoenix-node:1.0` docker image](https://hub.docker.com/r/joeellis/elixir-phoenix-node/).  This is a simple docker image I built with Elixir 1.4.2, Erlang 19.x, node 7.x, and `yarn` already installed. Any Docker image will do though - CircleCI even offers [pre-built Elixir images](https://circleci.com/docs/2.0/circleci-images/#elixir) if you'd rather not create your own.

The config also downloads a second docker image, [`postgres:9.6.2-alpine`](https://hub.docker.com/_/postgres/) to create a database container and with our app's database credentials ([see the official docker image for more information about supported env vars and options](https://hub.docker.com/_/postgres/).  Lastly, it sets a working directory folder called `app` in the CircleCI user's home directory.

Next, the build checks out our git repo, and restores any caches that may already exist:

{% highlight yaml %}
{% raw %}
- checkout
- restore_cache:
    keys:
    - v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
    - v1-mix-cache-{{ .Branch }}
    - v1-mix-cache
- restore_cache:
    keys:
    - v1-build-cache-{{ .Branch }}
    - v1-build-cache
- run: mix do deps.get, compile
{% endraw %}
{% endhighlight %}

The `restore_cache` and cache keys here make look funny to you. What are they and where do they come from?  In short, this is part of CircleCI's new caching mechanism, and before you read the rest of this article, I *highly* recommend you [read and understand their caching](https://circleci.com/docs/2.0/caching/) because you will need to understand it to create the best caching strategy for your own app.  After reading that, read below about the `save_cache` steps first and we'll cirle back to how this `restore_cache` stuff works in a minute.

{% highlight yaml %}
{% raw %}
steps:
  - save_cache:
      key: v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
      paths: "deps"
  - save_cache:
      key: v1-mix-cache-{{ .Branch }}
      paths: "deps"
  - save_cache:
      key: v1-mix-cache
      paths: "deps"
{% endraw %}
{% endhighlight %}

For our `deps` directory, we create three types of CircleCI caches:

1. A cache keyed against the branch name *and* the `mix.lock` checksum.
    - This cache is used for most commits to a branch.
    - It also uses a `mix.lock` checksum as part of its key. If new dependencies are added and the `mix.lock` file changes, this will make sure to cause a cache miss and force a recompilation of the new dependencies.
2. A cache keyed against just the branch name.
    - CircleCI uses this cache as a fallback if the first cache can't be found, usually when the `mix.lock` file has changed.
3. A cache with just a generic key of `v1-mix-cache`.
    - This cache is used if CircleCI can't find any of the above caches, like in the case of the first commit of a new branch. Also, if you are creating small commits often, then you'll find this cache is very useful at saving on compilation times between branches.

For our `build` directory, you can see it's very similar to the `deps` caching strategy:

{% highlight yaml %}
{% raw %}
- save_cache:
    key: v1-build-cache-{{ .Branch }}
    paths: "_build"
- save_cache:
    key: v1-build-cache
    paths: "_build"
{% endraw %}
{% endhighlight %}

One small difference is that we only use two types of caches as there is no 'lockfile' for the build directory to cache against.  This makes sense though, as we want your project to always recompile itself with the new changes made in each commit.

After you understand how the `save_cache` works, then the `restore_cache` keys in the steps above make more sense:

{% highlight yaml %}
{% raw %}
- checkout
- restore_cache:
    keys:
    - v1-mix-cache-{{ .Branch }}-{{ checksum "mix.lock" }}
    - v1-mix-cache-{{ .Branch }}
    - v1-mix-cache
- restore_cache:
    keys:
    - v1-build-cache-{{ .Branch }}
    - v1-build-cache
- run: mix do deps.get, compile
{% endraw %}
{% endhighlight %}

All we are doing here is declaring which of our saved caches to check and which order should check them.

Next, we use the exact same caching strategy to install our frontend `node_modules` using `yarn`:

{% highlight yaml %}
{% raw %}
- restore_cache:
    keys:
    - v1-yarn-cache-{{ .Branch }}-{{ checksum "assets/yarn.lock" }}
    - v1-yarn-cache-{{ .Branch }}
    - v1-yarn-cache
- run:
    working_directory: assets
    command: yarn install && yarn test
- save_cache:
    key: v1-yarn-cache-{{ .Branch }}-{{ checksum "assets/yarn.lock" }}
    paths: assets/node_modules
- save_cache:
    key: v1-yarn-cache-{{ .Branch }}
    paths: assets/node_modules
- save_cache:
    key: v1-yarn-cache
    paths: assets/node_modules
{% endraw %}
{% endhighlight %}

If you are using `npm`, the setup is roughly the same - instead of keying against a `yarn.lock` checksum, you would key against `package-lock.json`. I just prefer `yarn` as it is very fast and deterministic. Also, the latest `npm` (`5.3.0` at the time of this writing) was having some issues compiling correctly in production.

{% highlight yaml %}
- run: mix ecto.create && mix ecto.migrate
- run: mix phoenix.digest
- run: mix test
{% endhighlight %}

This last step should look familiar to anyone who has run a Phoenix application before.  Just create the database, create a digest for your assets if needed, and finally run `mix test`.

Hopefully, this rundown of CircleCI's 2.0 features helps someone out there. The config file looks long, but as you can see, the bulk of it is just repeated use of the same caching pattern. Give it a try, and if you run into trouble, free to [tweet at me](http://twitter.com/notjoeellis) or ping me on the [elixir-lang Slack channel](https://elixir-slackin.herokuapp.com/)!
