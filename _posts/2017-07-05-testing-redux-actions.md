---
layout: post
title: "Testing Redux: Async Action Creators"
date:  2017-07-05
description: "Async action creators are easier to test than you think."
tags:
  - react
  - redux
  - testing
---

One of the biggest (and often overlooked) advantages of Redux apps is how inherently testable they are. Redux apps are composed of mostly plain old JavaScript objects and functions, so testing rarely requires special tricks or methods.

And probably the most valuable tests you can write for a Redux app are action creator tests. Actions tend to collect most of the messy, complex code in a Redux app, and learning how to test them well can make future maintenance and refactoring easier for you and your team.

In this article, I'll explain the pattern I use for testing Redux actions ([asynchronous action creators](http://redux.js.org/docs/advanced/AsyncActions.html#async-action-creators) in particular) and hopefully show that testing even complex actions can be pretty easy!

### The Setup

#### Packages to install

For action testing, I use the following packages:

- [`mocha`](https://github.com/mochajs/mocha) - my JS testing framework of choice. If you don't use `mocha`, however, keep reading - the strategies in this article are not `mocha` specific and can work with any JS testing framework.
- [`nock`](https://github.com/node-nock/nock) - an HTTP mocking library. This is only used if your app is making HTTP requests (but at some point, your app probably will).
- [`redux-mock-store`](http://arnaudbenard.com/redux-mock-store/) - a simple, but effective Redux store mock.

#### Configuring nock

I usually create a base `nock` setup file in my app that I can import when needed.  It looks something like this:

{% highlight javascript %}

import nock from 'nock';

nock.disableNetConnect();

// Configure the base url to your endpoint here
export default nock("http://localhost/api/");

{% endhighlight %}

I recommend adding `nock.disableNetConnect()` as it will force disable all non-mocked HTTP requests during your test. This ensures your tests won't accidentally hit a real API endpoint during a test run, a bad practice that is both dangerous and slow.

### An Example Action Creator

Let's start with a look at a real world example, this set of action creators for fetching users from an API endpoint:

{% highlight javascript %}

const fetchUsersRequest = () => {
  return { type: Actions.FETCH_USERS_REQUEST }
}

const fetchUsersSuccess = (users) => {
  return {
    type: Actions.FETCH_USERS_SUCCESS,
    payload: users
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

Here, `fetchUsers` is the main action creator we want to test. When executed, it will send an HTTP request to the API, and dispatch two out of these three actions:

- `fetchUsersRequest` - an action used to represent a request in progress (handy if your UI needs to show some kind of loading state)
- `fetchUsersSuccess` - an action that fires if the API call succeeds
- `fetchUsersFailure` - an action that fires if the API call fails

We want our test to check that these action creators are all invoked in the correct order, with the correct data, for both successful and unsuccessful API calls. To do this, we'll do the following:

1. Setup a mock response for our HTTP API endpoint. If the action makes a request to `/users/123`, then we should make sure it hits our `nock` server and have `nock` return a mock response that we can control and test against.
2. Setup an ordered array of the actions we expect our action creator to dispatch and send to the reducers.
3. Dispatch the action creator function, assert it received our mock data, and assert it called all of our expected actions in the correct order.

### Testing The Happy Path

Given the strategy above, here is what testing the success path of `fetchUsers` might look like:

{% highlight javascript %}

import assert from "assert";
import mockStore from "redux-mock-store";
import nock from "../path/to/nockSetup";
import Constants from "../path/to/Constants";
import { fetchUsers } from "../path/to/UserActions";

describe("UserActions", function() {
  beforeEach(function() {
    this.store = mockStore({});
    this.usersData = [ {id: 1}, {id: 2} ];
  });

  context("#fetchUser", function() {
    it("can fetch users from the api", function() {
      nock.get("/users").reply(200, this.usersData);

      const expectedActions = [
        {
          type: Constants.FETCH_USERS_REQUEST
        },
        {
          type: Constants.FETCH_USERS_SUCCESS,
          payload: this.usersData
        }
      ];

      return this.store
        .dispatch(fetchUsers())
        .then(() => {
          assert.deepEqual(this.store.getActions(), expectedActions)
        })
        .catch(err => {
          throw(err)
        });
    });
  });
});

{% endhighlight %}

Let's break this test down into the important parts:

{% highlight javascript %}

beforeEach(function() {
  this.store = mockStore({});
  this.usersData = [ {id: 1}, {id: 2} ];
});

{% endhighlight %}

Here, we setup both our mock store and some fake data that we'll use later for our mock HTTP response.

{% highlight javascript %}

context("#fetchUser", function() {
  it("can fetch users from the api", function() {
    nock.get("/users").reply(200, this.usersData);
    ...

{% endhighlight %}

Next, we use `nock` to create a mock HTTP route for our Redux app to hit. This will return back our mock user data (`this.usersData`) whenever a successful GET request is made to the `/users` route.

{% highlight javascript %}

...
const expectedActions = [
  {
    type: Constants.FETCH_USERS_REQUEST
  },
  {
    type: Constants.FETCH_USERS_SUCCESS,
    payload: { users: this.usersData }
  }
];
...

{% endhighlight %}

Then we create an `expectedActions` array, an array of the actions we expect to be dispatched in the order we expect them to be dispatched in. The first object is the action created when `fetchUsersRequest` is invoked, and the second object is the action created when `fetchUsersSuccess` is invoked. Note: the `expectedActions` array here does not contain the action for `fetchUsersFailure` as we are only testing the success path right now.

{% highlight javascript %}

...
return this.store
  .dispatch(fetchUsers())
  .then(() => {
    assert.deepEqual(this.store.getActions(), expectedActions)
  })
  .catch(err => {
    throw(err)
  });
...

{% endhighlight %}

Finally, we kick off the test by dispatching our `fetchUsers` action creator using our mock store's `dispatch` function. `redux-mock-store` comes with a handy `store.getActions` function, so we can see which actions were called, and make sure it equals our `expectedActions` array.

### Testing The Not So Happy Path

We should also test that our action creators handle errors appropriately. Here is what testing the failure path of `fetchUsers` might look like:

{% highlight javascript %}

...

it("handle errors when fetching users from the api", function() {
  nock.get("/users").reply(500, this.usersData);

  const expectedActions = [
    {
      type: Constants.FETCH_USERS_REQUEST
    },
    {
      type: Constants.FETCH_USERS_FAILURE,
      payload: new Error()
    }
  ];

  return this.store
    .dispatch(fetchUsers())
    .then(() => {
      throw("This test should not have succeeded.")
    })
    .catch(err => {
      assert.deepEqual(this.store.getActions(), expectedActions)
    });
});

...

{% endhighlight %}

It has a very similar structure to the success test above. The main difference is our HTTP request mock now replies with a HTTP `500` status code (used for internal server errors) and our `expectedActions` array includes the results from the `fetchUsersFailure` action creator. Also, we run our `assert` in the `catch` portion of our promise (as we expect a failure here), and we throw an error in the success path in case our action creator incorrectly manages to succeed.

And that's it! You just learned how to fully test an async action creator for a Redux app

### Conclusion

With this testing strategy, even as your action creators become more complex, the testing strategy itself stays the same. They still follow the same pattern of creating a mock HTTP response, creating a list of expected actions, and comparing both to the result of the action creator function call. I recommend giving this testing strategy a try, especially if you've never tested your Redux actions before - I think you'll find them to be invaluable addition to your Redux app.


