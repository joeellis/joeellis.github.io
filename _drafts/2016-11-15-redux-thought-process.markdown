---
layout: post
title: "Things Learned Creating Redux Apps"
date:  2016-11-15
description: "Describing the patterns I use when building Redux apps"
categories: redux processes
---

*This post is part of a series to describe my thought processes when working in various languages and frameworks. Please do not consider these points as the best or only way to do things - these are simply my way to do things and I've found them to be useful enough to repeat and mention here.*

After using Redux for a year, I've found it to be a highly productive way to create SPA Javascript applications.  The only downside has been that unlike its alternatives, Redux is far less opinionated in how to best structure an app.  I personally prefer this freedom, but it does mean there is more of a learning curve.

### State

Planning the shape of your state upfront will often save you time in the long run. I've found state shapes that follow these concepts are the easiest to deal with long-term.

##### Avoid nesting when possible.

The native `Object.assign` method does not support deep merges yet, so handling deeply nested properties in your reducers starts can become annoying without using additional extra libraries or specialized functions. Those are possible avenues, to be sure, however some upfront planning about the state shape can usually help you skip over this issue altogether.

##### Only store raw data in the state.

People new to Redux will attempt to use the state as a vehicle to store any and all information they may think they'll need later. It's very tempting to do this, however doing so will often increase your app's complexity in the form of state bloat / repetitive properties, which in turn will trickle down and make your actions, reducers, and tests more complex.

Instead, focus on understanding the difference between is raw data and which is derivative, i.e. data can that be derived from other data. Then persist only the raw data you need for your app, deriving all other data you need for logic / presentational purposes using separate functions as the data works it's way through your React components. One of the biggest advantages of Flux / Redux is that its unidirectional flow creates a pipeline to easily allow us to transform data when we need it, so adding any data to your state should be carefully considered.

##### Choose a single source of truth.

React components come with [their own system for managing state](https://facebook.github.io/react/docs/react-component.html#state). In general, it's easier to maintain and debug an app using only a single source of truth where its data lives. Avoid mixing these when possible.

There are exceptions to this rule - for example, if you are creating a somewhat complex UI component that needs to persist a number of private properties that wont be used at a global level, then sure - you can likely get away without storing these in the Redux state. However, I've found it's better to stick to the general rule and approach these on a case by case basis.

### Actions

##### Standardize Action Payloads

It is really useful when working with a team to have an agreed upon object shape for your actions. You'll end up code that's much easier to maintain, test, and work in - I highly recommend adopting some kind of standard with your team. I personally use the [FSA spec](https://github.com/acdlite/flux-standard-action) because it's straightforward and simple to understand, however I think anything works as long as it's consistent between actions.

### Reducers

##### `createNewObject` Helper



### Component Architecture

##### Containers & Presentational Components

The most useful concept I've come across when researching how to build stable, maintainable redux apps is the [container & presentational component](http://redux.js.org/docs/basics/UsageWithReact.html#presentational-and-container-components) as described by Dan Abramov and the official Redux documentation. The docs do a great job at breaking down the concept with examples, so I will not describe it more here - however, please know that it is a solid concept worth emulating.

##### Intermediary Containers

The container / presentational component paradigm is great, but one common issue is understanding when a container should be made. Beginners will often create only one or two high-level containers for their app, and use that to fetch everything they need from the state for the children components. This usually results in props being passed through multiple components before they are actually used. As your app grows, this becomes a problem, as even small changes to your props (like renaming them) can involve changing many other non-related components just to get it to work - a code smell that something is not quite right.

If you notice this issue, consider introducing intermediary containers between components in this chain. That way you can access the state at that point to fetch and create props for the lower-level children and no longer need to pass everything through a large inheritance chain of components. Breaking up the chain in this way will definitely help lower your maintenance time & costs in the long run.

##### Importing Event Handler Functions

##### Separate Application Logic from Components

It's tempting to  data from the state and execute logic immediately in the `mapStateToProps` function or inside a component. This works fine for small things, but for larger apps, it has a bad tendency to couple your application logic inside the React / Redux framework.  For example, your code may start to evolve into something like this:

Instead, try to find create semantic functional modules that live outside of the React layer, for instance, inside of a `lib` folder.

##### Don't Be Afraid To Create Components

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
