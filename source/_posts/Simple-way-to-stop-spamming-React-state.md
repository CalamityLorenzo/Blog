---
title: Simple way to stop spamming React state in from child components
date: 2019-02-10 13:32:56
categories:
- Software
    - Typing
- Hacks
    - Basic
tags: 
- React
- Typescript
- Javascript
---

React has a strict approach to sharing state that goes in one direction, and one direction only: Down.
It passes from parent to child, and from React's perspective it doesn't want it back. If you need to change the state, you replace it. But how to get changes from the child component back up the tree again?

The typical approach is to use callbacks. You register a callback in the child component, that calls a method in the parent which updates the component state, and it works absolutely fine.

## Example
In the following codepen, we have two components, `ParentContainer` and a `ChildTextBox`.

<p class="codepen" data-height="265" data-theme-id="0" data-default-tab="js,result" data-user="calamitylorenzo" data-slug-hash="WPzjyW" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid black; margin: 1em 0; padding: 1em;" data-pen-title="WPzjyW">
  <span>See the Pen <a href="https://codepen.io/calamitylorenzo/pen/WPzjyW/">
  WPzjyW</a> by CalamityLorenzo (<a href="https://codepen.io/calamitylorenzo">@calamitylorenzo</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

When you type in the textbox the state of the parent is updated. With every single keypress you change the parent state, and React will happily handle it. 
If you open your browser console window you can see the new state being splashed across the screen. For every single keypress.

## So what's the problem?

Your application may not need to be aware of every single change of state. There are a lot of cases where all that matters is that state *HAS* been changed, and is now finished.
For instance, I've been working on a search application. User types into the search box, and it automatically makes a REST call to a search service. This is an expensive process calling to an external service. I'd rather keep it to a minimum.
There is no reason for that call to the search service to happen on every keypress. That's going to get my application rate-limited, ban hammers coming down, stern emails etc. More importantly it does not make a good user experience. If my user typed 'Joe Smith', that would be at least 9 searches and you can't gurantee the order of  REST query results. How do you work out which result is the correct search? If the user had typed 'Joe smithas' then deleted the mistaken characters, then the successful search had to be done twice, and filtering all this is going to take ages (for a user).

## A simple solution
Don't spam the parent state. React is built to take it on the chin, your application needs a little more finesse.
In the example we are using, you only care when the user finshes typing, so you can get the completed message to send to a service or do some other timebound action.
A simple way of achieving that is to apply a limiter. This will delay passing new versions of the state until a certain threshold has been exceeded.
In more concrete terms, we don't inform the parent state that the message has been updated until at least 750ms since the last known keypress.

<p class="codepen" data-height="265" data-theme-id="0" data-default-tab="js,result" data-user="calamitylorenzo" data-slug-hash="KJoqQN" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid black; margin: 1em 0; padding: 1em;" data-pen-title="KJoqQN">
  <span>See the Pen <a href="https://codepen.io/calamitylorenzo/pen/KJoqQN/">
  KJoqQN</a> by CalamityLorenzo (<a href="https://codepen.io/calamitylorenzo">@calamitylorenzo</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

The new class `Limiter` is simply a static wrapper around [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout) with a `CancelToken` that's just an alias for number, this is all we need to make a mechanism. A timeout to wait, and a token so we can cancel the previous action. If you don't cancel the action, then the function and it's arguments are  called.

In the `ChildTextBox` class I've added state, that's what will soak up all your keypress and calls until you are ready to send the new message back. I've also added a `CancelToken`, we maintain the most recent version, and pass it in to the Limiter when we update the message.
In the `_onChange` event,  the new message text arrives, we set the `ChildTextBox.state` to the new value (This guarantee that we are always keeping upto date), then make a call to  `Limiter.Execute`. The token cancels the previous call (setTimeout can cancel any previous calls, order is not important), and then pushes the newest one returning a new CancelToken in the process.

Once you have finished typing, you should see a pop-up message box. If you took out the limiter, it would be displayed after every kepress instead...That would be sub-optimal.

I like this method, it's simple (two variables and a call), idiomatic with `window.setTimeout` coupled with React's own very robust state handling. It's also very extensible. Any interface that defines a function means you have a hook to do anything else. The fact that it is native Javascript call, means no need to polyfill promises, for older browsers, and Typescript wont need to  build crazed statemachines to handle them.

