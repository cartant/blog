---
title: "RxJS: Avoiding Unbound Methods"
description: "Using a TSLint rule to avoid passing unbound methods"
date: "2018-06-30T07:20:27.713Z"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title.jpeg"
slug: "/@cartant/rxjs-avoiding-unbound-methods-fcf2648a805"
---

![Mops](title.jpeg "Photo by pan xiaozhen on Unsplash")

I often see code that looks a little like this:

```ts
return this.http
  .get<Something>("https://api.some.com/things/1")
  .pipe(map(this.extractSomeProperty), catchError(this.handleError));
```

Which seems fine.

Well, it is fine — as long as the `extractSomeProperty` and `handleError` methods do not depend upon the `this` context in their implementations.

## Why might this be a bad thing?

When unbound methods are passed to RxJS, they will be invoked with an unexpected context for `this`. If the method implementations don’t use `this`, they will behave as you would expect.

However, there are a number of reasons why, as a general rule, you might want to avoid passing unbound methods:

- If you are in the habit of passing unbound methods, it’s a near certainty that you will, eventually, introduce a bug by passing, as unbound, a method that uses `this` in its implementation. Maybe you’ll have a test that picks it up; maybe you won’t.
- Passing unbound methods is something that’s often done with error handlers. And the unit testing of code paths for errors is often neglected.
- The callbacks passed to `subscribe` — and some operators — are invoked with a `SafeSubscriber` instance as the context. That can make debugging any context-related issues more complicated, as `this` will not be `undefined`; it will be a valid object instance — just not the instance you were expecting.
- If a method’s implementation is changed to use `this`, anywhere it’s passed as unbound will have to be changed, too. Again, this is something that often happens with error handling: an initial implementation that writes to the console might be changed to instead report the error to a service — accessed via the `this` context.

## What are the alternatives?

If the method does not use `this`, it can instead be written as a static method or as a stand-alone function — rather than a method.

If the method does use `this`, it can be passed via an arrow function:

```ts
return this.http.get<Something>("https://api.some.com/things/1").pipe(
  map((s) => this.extractSomeProperty(s)),
  catchError((e) => this.handleError(e))
);
```

Or `bind` can be called to explicitly bind `this`:

```ts
return this.http
  .get<Something>("https://api.some.com/things/1")
  .pipe(
    map(this.extractSomeProperty.bind(this)),
    catchError(this.handleError.bind(this))
  );
```

At the moment, if `bind` is called, the type information is lost, but with the merging of [this PR](https://github.com/Microsoft/TypeScript/pull/24897) from Anders Hejlsberg, a future release of TypeScript will, eventually, have a strongly-typed `bind`.

## Using a TSLint rule

[TSLint](https://palantir.github.io/tslint/) is a fantastic tool for enforcing these sorts of statically-analysable rules. I’ve added an `rxjs-no-unbound-methods` rule to [`rxjs-tslint-rules`](https://github.com/cartant/rxjs-tslint-rules) — a suite of TSLint rules that can be used to ensure an RxJS codebase is clean and hygienic.

So if passing unbound methods is something you’d like to avoid, too, install the package and switch on the rule.

TSLint includes a built-in [`prefer-function-over-method`](https://palantir.github.io/tslint/rules/prefer-function-over-method/) rule which is related, but is a little different. That rule exists to prevent methods that don’t use `this` from being written in the first place — which might or might not be what you want.
