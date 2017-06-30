---
layout: post
title: "Testing Redux: Actions"
date:  2017-06-20
description: "Part of a series showing how to fully test a Redux app"
tags:
  - react
  - redux
  - testing
---

*This is part of a larger series of posts to show how each component of a Redux app can be easily unit tested.*

### Testing Redux: Action Creators

In this article, I will go over a few strategies I use to unit test Redux actions.  Redux actions and action creators, in my opinion, are one of the most important things to test in a Redux app because it's most likely the place where the logic and code can get a little messy.

### The Setup

To reliably test actions, I use the following packages:

- A JS testing framework - I usually use [`mocha`](https://github.com/mochajs/mocha) for testing. However, if you don't use `mocha`, keep reading - most ideas in this article can still translate into the testing framework you use.
- [`nock`](https://github.com/node-nock/nock) - an HTTP mocking library. This is only needed if your app is making HTTP requests (but chances are high that it is!)
- A mock Redux store - I use [`redux-mock-store`](http://arnaudbenard.com/redux-mock-store/) as it is a very simple, but effective store mock. It comes built-in with a `dispatch` function and will record all actions it has dispatched.

### How Redux Action Creator Testing Works

Actions can seem tricky to test as they require the use of two mocks to work, one for the HTTP call and one for the store. But in practice, it's quite simple.  Our action creator testing strategy will follow this path:

1. We create a mock response for a given HTTP API call. For example, if the action makes a call to `/users/123`, then we want to capture that request and create a mock response we can control and test against.
2. We setup an array of the actions we expect this action creator to dispatch in the lifetime of its execution. This way we can ensure all the actions we care about are called in the correct order.
3. We use our mock Redux store to actually execute the action creator, and assert that the data returned and the order of actions is correct.


### Example Test

Let's take a look at a real world example - a set of action creators for fetching a user from an API endpoint:

{% highlight javascript %}

const fetchUsersRequest = () => {
  return { type: Actions.FETCH_USERS_REQUEST }
}

const fetchUsersSuccess = (users) => {
  return {
    type: Actions.FETCH_USERS_SUCCESS,
    payload: { users }
  }
}

const fetchUsersFailure = (err) => {
  return {
    type: Actions.FETCH_USERS_FAILURE,
    payload: err
  }
}

export const fetchUsers = () => {
  return (dispatch) => {
    dispatch(fetchUsersRequest());

    return fetch("/users")
    .then(({ users }) => {
      dispatch(fetchUsersSuccess(users));
    })
    .catch((err) => {
      dispatch(fetchUsersFailure(err));
    })
  }
}

{% endhighlight %}

Here, `fetchUsers` is the main action creator we are interested in testing. During its lifetime, it will make an HTTP call to the API, and dispatch three other action creators:

- `fetchUsersRequest` - an action used to represent a request in progress (handy if your UI needs to show some kind of loading state)
- `fetchUsersSuccess` - an action that fires if the API call succeeds
- `fetchUsersFailure` - an action that fires if the API call fails

We want to test both the success and failure path of `fetchUsers`.  Let's first take a look at testing the success path:

{% highlight javascript %}

import assert from "assert";
import mockStore from "redux-mock-store";
import nock from "../path/to/nockSetup";
import Constants from "../path/to/constants";
import { fetchUsers } from "../path/to/UserActions";

describe("UserActions", function() {
  beforeEach(function() {
    # We setup both our store and some fake data for our tests
    this.store = mockStore({});
    this.usersData = [ {id: 1}, {id: 2} ];
  });

  context("#fetchUser", function() {
    it("can fetch users from the api", function() {
      # We mock out the HTTP call so that when a GET request is made to the
      # `/users` route, it will reply back with a HTTP 200 status code
      # and our test data
      nock.get("/users").reply(200, this.usersData);

      # An array of the actions we expect to be dispatched when
      # `fetchUsers` is run
      const expectedActions = [
        {
          type: Constants.FETCH_USERS_REQUEST
        },
        {
          type: Constants.FETCH_USERS_SUCCESS,
          payload: { users: this.usersData }
        }
      ];

      # We use the mockStore to dispatch our `fetchUsers` action creator
      # and assert
      return this.store
        .dispatch(fetchUsers())
        .then(() => {
          assert.deepEqual(this.store.getActions(), expectedActions)
        })
        .catch(err => {
          throw(err)
        });

      return this.store.dispatch(fetchUser()).then(() => assert.deepEqual(this.store.getActions(), expectedActions));
    });
  });
});

{% endhighlight %}



### Useful tip

Make sure to enable `nock.disableNetConnect()` in your test suite
