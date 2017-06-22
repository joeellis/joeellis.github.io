---
layout: post
title: "Redux Architecture Guidelines"
date:  2017-06-20
description: "Describing patterns I use when building Redux apps."
tags:
  - react
  - redux
---

I've written many Redux apps over the years, and it is by far my favorite JS framework. The only downside is, unlike other frameworks, Redux is far less opinionated in how to structure an app. I prefer this freedom, but it does make for a steeper learning curve, especially if you're new to Redux.  So I decided to write up some of the higher level thinking and structure I've picked up and often use when building a Redux app. Hopefully it comes in handy for someone out there.

### State

##### Plan your state shape

In terms of saving time down the road, planning the structure of your state object upfront is most valuable thing you can do for your app. A poorly formed state object will make your app difficult to maintain and is avoidable with some planning. I run through this quick checklist when planning out state objects:

- How will it store multiple resources from an API (users, accounts, items, etc)?
- How will it handle loading states (showing loading spinners when fetching /updating data)?
- How will it handle showing and clearing of UI success and error notifications?
- Does it feel consistent and predictable? Could another team member easily work with it?
- Is it easy to access data within it? Does it nest properties unnecessarily?
- Is it serializable? Could it easily be stored away in localstorage or in a database?
- Are there any properties you could pull from the URL instead of in the state?
- Is there any duplicated data in here? If so, is that really needed?

There are **many** different ways to answer these questions - it depends on your app. But in my experience, having at least an answer for each will save you time in the long run.

##### Avoid nesting state objects

Some Redux apps have deeply nested state structures, i.e. shapes that look like this:

{% highlight javascript %}
{
  foo: {
    bar: {
      baz: {
        qux: ...
      }
    }
  }
}
{% endhighlight %}

This often happens when we work with relational data as it feels natural to use nesting to represent those relationships. Unfortunately, nested data structures create complexity. At the component level, you'll have to reach even deeper into the state to get certain information. And at the reducer level, merging new data into your state will become far more complex. On top of all of that, nested data can even cause performance issues with React / Redux itself.

Consider instead to flatten and [normalize](http://redux.js.org/docs/recipes/reducers/NormalizingStateShape.html) your state shape. In Redux land, the shallower the nesting, the easier it is is to fetch and update state data in your app. Normalized states help solve the problems listed above, and make your state much more flexible overall.

##### Storing only raw data in the state

It's tempting to use Redux's state as a vehicle to store any and all information you think you may need later. Yet, doing so will increase your app's complexity in the form of state bloat and redundant properties. This, in turn, increases complexity in your actions, reducers, and tests. So what should and shouldn't be stored?

In Redux apps, there are really two types of data. The first is raw data, data your app requires to run. User data fetched from an API is an example of raw data - without it, your app won't have the information it needs to run. The second is derived data, or data created from other existing data. Using the `firstName` and `lastName` properties to display a user's name as `Jane Doe` is an example of derived data.

I recommend persisting **only** raw data in your state. It helps reduces state bloat and makes it easier to reason about what data is important in your app. All other derived data should be created using functions that accept that raw data from the state return back the information you need.

Before adding something new to the state object, ask yourself this question, "Can I create this from data that already exists in the state?"  If the answer is "yes", then create that data with a function. If the answer is "no", then you may have a good case to add this data to the state.  You may be surprised over time how often the answer is "yes."

##### Prefer Redux state over React state

React comes with [its own system for managing state](https://facebook.github.io/react/docs/react-component.html#state) inside of components. In a Redux app, though, prefer to use Redux's state for the majority of your app data and inter-component communication. It is overall much easier to reason about your app when there is one accepted way for components to set and access state, especially if you are working within a team.

Note that there are reasonable exceptions to this guideline. It can be beneficial for complex UI components to persist local properties using React component state, especially when those properties aren't globally important to the app. When doing this, just try to keep that React state management localized to that component. Using two separate state systems too much, especially for inter-component communication, is likely to cause confusion for the developer after you.

### Actions

##### Standardize action payloads

When working with a team, having a standard object shape for your actions is very helpful. Doing so reduces [bikeshedding](http://bikeshed.org/) and creates maintainable and testable code. I highly recommend adopting some kind of standard with your team. I use the [Flux Standard Action spec](https://github.com/acdlite/flux-standard-action) because it is straightforward and simple to understand. But whatever you use, make sure it's consistent and easy to work with.

##### Ensure action creators are composable

Many example apps and tutorials I run across use simple action creator functions when teaching Redux concepts. This is great for illustrating a point, but real world apps are complex. It's inevitable that you will need to compose higher-level complex actions, preferably from existing action creators you have already written.

Start a habit of making sure all your action creator functions are composable in some way. It's a simple rule that really pays off when you need it. I personally wrap each action creator in a [promise](https://github.com/then/promise) so they can be easily chained together using the `then` function.

### Component Architecture

##### Containers & presentational components

The most useful concept I've come across for building stable and easily maintainable Redux apps is the [container & presentational component](http://redux.js.org/docs/basics/UsageWithReact.html#presentational-and-container-components) paradigm as described by Dan Abramov in the official Redux documentation. I will not dive into it here as the docs already do a great job at explaining the concept with great examples. But understanding this paradigm may be one of the most useful things you can learn about in Redux land.  It is very difficult to maintain and iterate on an app of even moderate complexity without it. Learn it well.

##### Use intermediary containers

While the container / presentational component paradigm works, it's not always clear when containers should be introduced. I've seen (and written) apps with a single top-level container that fetches the whole world and then passes down everything to its component's children and their children's children. This results in props 'passing through' multiple components before they are ever even used. As your app grows, this becomes an annoying problem as even simple changes, like renaming props, involves changing many other non-related components. Definitely a code smell that something is not right.

Instead, create containers when you notice multiple props 'passing through' multiple components. There is no need to pass props from one end to the other when a container in the middle can access the state and create those props for you. Intermediary containers also have added benefits, such as encapsulating sections of your component tree making their children easier to maintain and test. Don't be afraid to use them if the situation calls for it.

### There are No Rules

All the guidelines I've listed are just patterns I've found worth repeating. However, do not consider any these points as the *only* way to do things. After all, one of biggest advantages of Redux is its free form structure, so know when you should 'break' the rules and try something new.
