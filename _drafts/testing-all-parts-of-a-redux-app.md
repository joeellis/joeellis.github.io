---
layout: post
title: Testing All Parts of a Redux App
---

### Testing

Testing in React right now is very interesting and still changing. As of the time of this writing, I've found these strategies have helped quite a bit:

##### Shallow vs Mount Rendering

React has two ways to render a component within a test:

- Shallow rendering - React will render the component in its own virtual DOM, and only render it one-level deep (meaning just the component itself, no children).
- Mount rendering - React will render a component and mount it onto some kind of DOM, like `jsdom`. It will render the component, as well as all of the component's children.

For unit tests, prefer to shallow render components as much as possible. It's not only far faster method than mounting (as it only uses React's virtual DOM), but it ensures the component is rendered independent from other components. There is one exception to this rule though, which is that shallow rendering does not currently support `refs`, so you will need to mount components where you need to test a `refs` property.

Please do note that fully rendering a component is still useful, especially if you are attempting to do a fuller end-to-end type integration test for your apps. It's unlikely though that these will make up the majority of your test suite though.

##### Use Enzyme

Airbnb has a fantastic library called [`enzyme`](https://github.com/airbnb/enzyme) that provides a far better API than React's TestUtils for dealing with and accessing component properties during testing. I personally am not a fan of using external packages when I can help it, but `enzyme` is a must-have for any of the React projects I work on.

##### Promises

##### Testing Actions

When testing actions, you can use a mock store ([like this one](https://github.com/arnaudbenard/redux-mock-store)) to dispatch actions and test against the end result.  Combined with an http mocking library like [nock](https://github.com/node-nock/nock), you can create some really great functional tests for whatever combination of test cases you can think of.
