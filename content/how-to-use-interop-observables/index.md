---
title: "RxJS: How to Use Interop Observables"
description: A complete guide to interop observable sources in RxJS
date: "2020-03-13T17:49:00+1000"
categories: []
keywords: []
cardImage: "./title-card.jpeg"
---

![Plug](title.jpeg "Photo by Hayley Maxwell on Unsplash")

RxJS is all about observables. It's a package that contains observable source creators and operators to manipulate them, but observables aren't just an RxJS abstraction.

## Observables

In 2009, [Erik Meijer](https://twitter.com/headinthebox) created a [reactive programming](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) framework for .NET named Reactive Extensions — also known as ReactiveX or Rx.

Rx.NET introduced the `Observable` type — a combination of the observer and iterator patterns — along with an implementation [contract](http://reactivex.io/documentation/contract.html) that specified how observables and their subscribers should interact.

Around that time, Rx was ported to other languages and platforms. [Matt Podwysocki](https://twitter.com/mattpodwysocki) authored the JavScript port — [RxJS](https://github.com/Reactive-Extensions/RxJS) — and its early versions were a little different to the package that we use today (the above link is to a separate, now-archived repository). They were closer to the .NET `Observable` type:

- observers had `onNext`, `onError`, `onCompleted` methods; and
- `subscribe` returned a [disposable](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/disposables/disposable.md).

In 2015, [Jafar Husain](https://twitter.com/jhusain), [Kevin Smith](https://twitter.com/zenparsing), [et al.](https://github.com/tc39/proposal-observable/graphs/contributors) submitted a TC39 proposal for the introduction of an [`Observable`](https://github.com/tc39/proposal-observable) type to the ECMAScript standard library. The proposal's `Observable` type is the one with which we are familiar — observer's have `next`, `error` and `complete` methods; and `subscribe` returns a `Subscription` — because RxJS version 5 was a re-write — by [Ben Lesh](https://twitter.com/BenLesh), [Paul Taylor](https://github.com/trxcllnt), [et al.](https://github.com/ReactiveX/rxjs/blob/5.0.0/package.json#L115-L140) — to bring RxJS into line with the proposal.

Since 2017, the TC39 proposal has been [stalled at stage 1](https://github.com/tc39/proposals/blob/c8b78027c9d8301e7243012bd78bed61ea075505/stage-1-proposals.md) — for [reasons](https://github.com/whatwg/dom/issues/544#issuecomment-351607779) given by Jafar, the proposal's champion — but, recently, some members of the RxJS core team have spoken with [Daniel Ehrenberg](https://mobile.twitter.com/littledan) about the 'resurrection' of the TC39 proposal.

What will come of this — or of the parallel [WHATWG proposal](https://github.com/whatwg/dom/issues/544) — I don't know, but the point of this historical recap has been to highlight that observables aren't just an RxJS abstraction. There are [multiple `Observable` implementations](https://github.com/tc39/proposal-observable/blob/d3404f06bc70c7c578a5047dfb3dc813730e3319/README.md#implementations) and those implementations need to be able work with one another. They need to be able to interoperate. How that is achieved is detailed in the proposal and that's what we're going to look at in this article.

## Interoperability (interop)

The proposal defines a well-known symbol — `Symbol.observable` — for identifying observable instances. Within an observable implementation, it's possible to use `instanceof` to determine whether or not an instance is an observable, but for interop purposes, it's necessary to use a well-known symbol.

Observable instances implement a `Symbol.observable` method — which returns an object that implements `subscribe`. Typically, the method will return the instance itself and in RxJS, the method's implementation looks like this:

```ts
/**
 * @method Symbol.observable
 * @return {Observable} this instance of the observable
 */
[Symbol.observable]() {
  return this;
}
```

JavaScript uses a bunch of [well-known symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol#Well-known_symbols) to identify language behaviours. For example, the presence of a `Symbol.iterator` method — that returns an [iterator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators) — is how iterables are identified:

```js
const iterable = {
  [Symbol.iterator]() {
    let value = 0;
    return {
      next() {
        return value < 3
          ? { value: value++, done: false }
          : { value: undefined, done: true };
      }
    };
  }
};
```

Once identified as iterable, the instance can be used anywhere an iterable is expected — such as in an array expression that uses the spread syntax:

```js
console.log([...iterable]); // [0, 1, 2]
```

To determine whether an instance is observable — and to subscribe to it, if it is — `Symbol.observable` could be used like this:

```ts
if (typeof instance[Symbol.observable] !== "function") {
  throw new Error("Not observable");
}
const observable = instance[Symbol.observable]();
observable.subscribe(value => console.log(value));
```

Consuming interop observables in RxJS is a little simpler.

RxJS declares an [`InteropObservable`](https://github.com/ReactiveX/rxjs/blob/28e317e4b1c050a76abb57e798ab419fcbe8ed47/src/internal/types.ts#L57) type which is included in the [`ObservableInput`](https://github.com/ReactiveX/rxjs/blob/28e317e4b1c050a76abb57e798ab419fcbe8ed47/src/internal/types.ts#L52) type's union — via the [`SubscribableOrPromise`](https://github.com/ReactiveX/rxjs/blob/28e317e4b1c050a76abb57e798ab419fcbe8ed47/src/internal/types.ts#L37) type — so interop observables can be used anywhere an `ObservableInput` is accepted. Which means interop observable instances can be passed to [`from`](https://github.com/ReactiveX/rxjs/blob/28e317e4b1c050a76abb57e798ab419fcbe8ed47/src/internal/observable/from.ts#L6), like this:

```ts
from(instance).subscribe(value => console.log(value));
```

It's not always necessary to call `from` on interop observable instances. Because interop observables are valid `ObservableInput`s, they can also be passed directly to factory functions like [`concat`](https://github.com/ReactiveX/rxjs/blob/28e317e4b1c050a76abb57e798ab419fcbe8ed47/src/internal/observable/concat.ts#L20) or returned from `project` functions passed to operators like [`concatMap`](https://github.com/ReactiveX/rxjs/blob/28e317e4b1c050a76abb57e798ab419fcbe8ed47/src/internal/operators/concatMap.ts#L5).

Every RxJS observable implements `Symbol.observable`, so any package that's able to consume an interop observable is able to consume an RxJS observable. That means [Bacon.js](https://github.com/baconjs/bacon.js/), [Callbags](https://github.com/callbag/callbag), [Kefir](https://github.com/kefirjs/kefir) and [xstream](https://github.com/staltz/xstream) can all consume RxJS observables.

## Common interop problems

Using `Symbol.observable` as the interop point works, but it does introduce some problems.

The first problem is that it's just a proposal, so `Symbol.observable` won't be defined when an application starts. It's up to the developer to define it — i.e. to [polyfill](<https://en.wikipedia.org/wiki/Polyfill_(programming)>) it.

RxJS used to polyfill `Symbol.observable`, but its doing so was [removed in version 6](https://github.com/ReactiveX/rxjs/pull/3387). One of the reasons for this is that polyfilling `Symbol.observable` is best left to the developer.

If multiple packages depending upon `Symbol.observable` are used and if each package attempts to polyfill the symbol, the application behaviour can depend upon the order in which the packages are loaded and that can expose subtle bugs. Developers can avoid the problem by polyfilling the symbol themselves, as early as possible in the bootstrapping of the application:

```ts
if (!Symbol["observable"]) {
  Object.defineProperty(Symbol, "observable", {
    value: Symbol("observable")
  });
}
```

Another problem is that it's not possible to [ponyfill](https://github.com/sindresorhus/ponyfill) `Symbol.observable` because of the way that TypeScript deals with well-known symbols.

[Declaration merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) can be used to declare a type for `Symbol.observable` — based on TypeScript's [`Symbol.iterator` declaration](https://github.com/microsoft/TypeScript/blob/ad8d3d90a5a8cc6a3b088a7b717e002d755e89f3/lib/lib.es2015.iterable.d.ts#L23-L29) — like this:

```ts
declare global {
  interface SymbolConstructor {
    readonly observable: symbol;
  }
}
```

However, to be assignable to RxJS's `InteropObservable` type, TypeScript requires `Symbol.observable` to be used in the method's computed property. That is, an interop observable has to be declared like this:

```ts
const instance = {
  [Symbol.observable]() {
    return /* subscribable */;
  }
};
const wrapped = from(instance); // OK
```

If the interop observable is declared like this, it won't be assignable to `InteropObservable`:

```ts
const observable = Symbol.observable;
const instance = {
  [observable]() {
    return /* subscribable */;
  }
};
const wrapped = from(instance); // ERROR
```

The reason for this is that the use of a property of `Symbol` has a specific implication for TypeScript: it doesn't seem to be the case that it has a `unique symbol` type; rather, it appears to be treated differently solely because it is a property of `Symbol`. That means that `Symbol.observable` cannot be ponyfilled and that the `observable` symbol exported by RxJS cannot be used when declaring interop observables.

This is another reason why developers should polyfill `Symbol.observable` themselves and should ensure that it's polyfilled before any packages that depend upon it are loaded.

The last problem is that implementing `subscribe` can be a little inconvenient, as it can be passed either an observer or separate `next`, `error` and `complete` callbacks.

Implementing `subscribe` is necessary in situations in which interop observables are used to expose functionality that can be used either with or without an observable implementation. For example, here's a trivial greeter object that can be used directly — by calling `greet` — or can be used an an observable source:

```js
const greeter = {
  [Symbol.observable]() {
    return {
      subscribe(nextOrObserver, error, complete) {
        return greet(greeting => {
          if (typeof nextOrObserver === "function") {
            nextOrObserver(greeting);
          } else {
            nextOrObserver.next(greeting);
          }
        });
      }
    };
  },
  greet(callback) {
    const id = setTimeout(() => callback("Hi!"), 1e3);
    return () => clearTimeout(id);
  }
};
```

This use of the observable interop point can appeal to package authors, as it offers a way of creating observable sources without a dependency on RxJS — or on another observable implementation.

[Christoph Guttandin](https://github.com/chrisguttandin)'s [`subscribable-things`](https://github.com/chrisguttandin/subscribable-things) package uses this interop observable approach to wrap DOM APIs — like [`matchMedia`](https://developer.mozilla.org/en-US/docs/Web/API/Window/matchMedia) — so that they can be used as observable sources with RxJS, Bacon.js, Callbags, Kefir or xstream.

## rxjs-interop

To make some of these problems a little less annoying, I created a package: [`rxjs-interop`](https://github.com/cartant/rxjs-interop). It has no dependency on RxJS and includes a couple of helpers — `patch` and `toObserver` — that make dealing with interop observables like `greeter` a little easier.

`patch` can be passed an instance or a class constructor and will ensure that even if the application developer hasn't polyfilled `Symbol.observable`, the interop observable will still 'play nice' with RxJS.

The implementation of `patch` was motivated by Angular. There are a large number of Angular developers and — as Angular itself depends upon RxJS — a helper like `patch` that ensures interop observable packages — like `subscribable-things` — will 'just work' with Angular makes things much simpler.

`toObserver` can be passed the arguments received by `subscribe` and can be relied upon to always return an observer.

Using the two helpers, `greeter` looks like this and will be compatible with RxJS regardless of whether or not the application developer has polyfilled `Symbol.observable`:

```js
import { patch, toObserver } from "rxjs-interop";
const greeter = patch({
  [Symbol.observable]() {
    return {
      subscribe(...args) {
        const observer = toObserver(...args);
        return greet(greeting => observer.next(greeting));
      }
    };
  },
  greet(callback) {
    const id = setTimeout(() => callback("Hi!"), 1e3);
    return () => clearTimeout(id);
  }
});
```

Observable interop isn't something that you'll often need to deal with, but, hopefully, this article — and the `rxjs-interop` package — will be helpful should the need arise.
