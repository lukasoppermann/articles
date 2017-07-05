---
title: A case against Polymer and in favor of open web components
tags: polymer, web-components
author: Lukas Oppermann
category: technology
description: Polymer is bad for the web component eco system, but this can be changed.
preview: Polymer is supposed to accelerate to adoption of web components but it also segregates web components, which are Polymer exclusive and this not in the spirit of web components which are supposed to be dependency free and reusable for everybody.
---

{.o-article__introduction}
> Polymer is the de facto library for building web components today. It is hard to find a web component that is not build with it. However this goes against the very idea of web components as dependency free, interoperable modules which can be dropped into any web project. But before we dig into this let me clarify what the Polymer project actually is.

## Polymer the "library"
Polymer is a web component **library** by Google. According to the [polymer projects website](https://www.polymer-project.org/1.0/):

>> The Polymer Project helps you deliver amazing user experiences by unlocking the full potential of the web platform.

Further on it is stated that:

>> Polymer sprinkles a bit of sugar over the standard Web Components APIs, making it easier for you to get great results.

So to sum up, Polymer is a sugaring layer on top of web components, that tries to make creating web components easier to work with.

## What are web components?

MDN has a very good [summary of the idea behind web components](https://developer.mozilla.org/en-US/docs/Web/Web_Components):
>> [...] reusable user interface widgets that are created using open Web technology. They are part of the browser, and so **they do not need external libraries** like jQuery or Dojo (see also [Wikipedia](https://en.wikipedia.org/wiki/Web_Components), [Google developer page](https://developers.google.com/web/fundamentals/getting-started/primers/customelements)).

TL;DR Web Components = (UI) components that can be reused without writing additional code and or using external libraries.

Why would you want this? Because currently if you find a very nice component, it might be react but you are using Angular so you can't use it. The solution web components deliver is to separate small components from frameworks and libraries to be used by everyone with any framework.

## The Problem
Polymer is a paradox in itself. By trying to get more people to use web components, they are actually betraying the very idea of web components: interoperability.

As of now you have a hard time finding a web component that is not tied to polymer. That means even if you are just using a single rating component you will need to include the polymer library.

What is much worse is that when a new version of polymer is release you will need to update all components. What if you have a well working component which is not updated? Do you include the old and the new polymer?

Google might say that Polymer is not a framework but a library. However you are tight in just the same.

## Is google evil?
I don't think so. I don't believe google wants to be the only company controlling web components. I rather think they wanted to show what is possible with web components and went a little overboard.

## What's the solution?

The ideal solution has three steps, that while adding a bit of work will be beneficial for all.

Firstly we need to start writing much more vanilla web components. Google should do this as well. I want to see form elements, sliders, buttons and more working without any dependencies, just like mozilla's vision above states.

This would improve current we components. For example, when you can't rely on polymer being there, you would make input elements work in any web form, not just in a special polymer form.

## Will we have to rewrite the same code for every component?
No, instead of creating one big library we could create many tiny packages that you bake into your component. This can be done using es6 modules with babel + gulp. This way google could be writing a module to sync attributes with class variables and everybody could use it.

## Can libraries like Polymer be a good thing?
I think Polymer should be called what it is, a framework. It's a framework which uses web components instead of building it's own component system.

This is pretty neat as it uses much native functionality as possible. This makes much sense for when you build your app. Logical components like routers that need to be more tightly integrated can be Polymer only. However let's start making our inputs, buttons and toggles dependency free so everyone can use them.
