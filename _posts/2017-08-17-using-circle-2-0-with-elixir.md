---
layout: post
title:  "Using Circle 2.0 with Elixir and Phoenix"
date:   2017-08-17
description: "Making the most of CircleCI 2.0 to create an automated CI for Elixir tests"
tags:
  - elixir
  - circleci
---

I recently converted an Elixir / Phoenix application to build it's CI jobs using CircleCI's new 2.0 version.  CircleCI 2.0 [boasts many new features](https://circleci.com/docs/2.0/#features) but the ones I was most interested in was their support for Docker as well as their advanced caching features.

The new Docker image support has helped save the most time as CircleCI no longer needs to compile Elixir, Erlang, and Node for each build. And their advanced caching features go a step further to cache our `build` and `deps` directories, saving us on the compilation time between jobs.  Overall these features helped us cut our build times in *half*.  icoIt did take some research through their docs and forums to create a complete CircleCI yaml file, so I wanted to write up what I did in case my example helps save time for other people.

To start, we'll work with a brand new Phoenix app that uses the following:

- Elixir 1.4.2, Erlang 19.x, and Node 7.x
- `yarn` to install npm libraries
- PostgreSQL for the database

Your stack may differ from the above, and that's ok.  I'll explain where those points are later in the article. Just know there may be a few things you'll have to change to make it work for your project, but the strategy outlined here should work for the majority of Phoenix projects.

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
          - POSTGRES_USER=phoenix
          - POSTGRES_PASSWORD=phoenix
          - POSTGRES_HOST=localhost
    working_directory: ~/repo
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

That looks like a lot, so let's look at this piece by piece and see what's going on.

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
          - POSTGRES_USER=phoenix
          - POSTGRES_PASSWORD=phoenix
          - POSTGRES_HOST=localhost
    working_directory: ~/repo
{% endraw %}
{% endhighlight %}

First we tell CircleCI that we'd like our build to run under a docker image called `joeellis/elixir-phoenix-node:1.0`.  This is a simple docker image I created that comes with Elixir 1.4.2, Erlang 19.x, node 7.x, and `yarn` already setup and installed.

It also downloads a second docker image, `postgres:9.6.2-alpine` to setup our database and our database credentials using environment variables ([see the official docker image for more information about supported env vars and options](https://hub.docker.com/_/postgres/).  Lastly, it sets a working directory folder called `repo` in the CircleCI user's home directory.

{% highlight yaml %}
{% raw %}
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
{% endraw %}
{% endhighlight %}

Next, the build runs through a set of initial steps to checkout our git repository, restore any previous caching we may have done, fetch and compile our dependencies, and save them again in CircleCI's cache.

The cache keys here make look funny to you. This is part of CircleCI's new caching mechanism, and before you read the rest of this article, I *highly* recommend you [read and understand their caching](https://circleci.com/docs/2.0/caching/) because you will need to understand it to figure out the best caching strategy for your own app.

For our `deps` directory, we create three types of caches:

1. A cache keyed against the branch name *and* the `mix.lock` checksum. This cache is used for almost all commits to a branch except the first one or if your commit causes a change in the `mix.lock` file.  Note: it's important to key off the `mix.lock` file as caches are shared between jobs and someone on your team could be working in a separate branch and running jobs with a different mix file from yours.
2. A cache keyed against just the branch name.  CircleCI uses this cache if the first cache can't be found, usually when the `mix.lock` file has changed.
3. A cache with just a generic key of `v1-mix-cache`.  This cache is used if CircleCI can't find any of the above caches, such as in the case of the first commit of a new branch. However, if you are making sure to make small commits often, then you'll find this cache is still quite useful at saving on compilation times.

For our `build` directory, you can see it's very similar to the `deps` caching strategy.  One small difference is that it only uses two types of caches as there is no cache against a lockfile.  This makes sense though, as most changes you make will require your project to be recompiled anyways.

After you understand how the cache saving works, then the `restore_cache` keys in the steps above make more sense.  It is simply the place we declare which caches to pull and which order to pull them in.

Next, we use the same strategy to install our frontend `node_modules` using `yarn`:

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

This last step should look familiar to anyone who has run a Phoenix application before.  Just create the database, create a digest for your assets if needed, and run `mix test`.

