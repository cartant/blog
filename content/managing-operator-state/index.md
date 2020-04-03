---
title: "RxJS: Managing Operator State"
description: How to avoid problems with operators that store state
date: "2019-02-11T20:34:01.280Z"
categories: []
keywords: []
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-managing-operator-state-2f20681df21d
---

![Shipping container](title.jpeg "Photo by Victoire Joncheray on Unsplash")

When [pipeable operators](/understanding-lettable-operators/) were introduced in RxJS version 5.5, writing user-land operators became much simpler.

A pipeable operator is a higher-order function: a function that returns another function. And the function that is returned takes an observable and returns an observable. So, to create an operator, you don’t have to subclass `Operator` and `Subscriber`. You just write a function.

Simple.

However, there are situations in which you need to take some extra care. In particular, you need to be careful whenever your operator stores internal state.

## An example

Let’s look at an example: a `debug` operator that logs received values and their indices to the console.

Our operator will need to maintain some internal state: the index — which will be incremented each time a `next` notification is received. A naive approach would be to store that state within the operator function. Like this:

```ts
import { MonoTypeOperatorFunction } from "rxjs";
import { tap } from "rxjs/operators";

export function debug<T>(): MonoTypeOperatorFunction<T> {
  let index = -1;
  // Let’s pretend that the map operator doesn’t exist and that we have to use
  // the tap operator and maintain our own internal state for the index, as the
  // purpose of our operator is to show that the behaviour depends upon where
  // the state is stored.
  return tap((t) => console.log(`[${++index}]: ${t}`));
}
```

However, this approach has a couple of problems that will effect surprising behaviours and hard-to-find bugs.

## The problems

The first problem is that our operator is not referentially transparent. A function is referentially transparent when it’s possible to replace the function call with the value that would be returned without changing the program’s behaviour.

Let’s look at what happens when our operator’s returned value is used to compose several observables:

```ts
import { range } from "rxjs";
import { debug } from "./debug";

const op = debug();
console.log("first use:");
range(1, 2).pipe(op).subscribe();
console.log("second use:");
range(1, 2).pipe(op).subscribe();
```

The program’s output is:

```text
first use:
[0] 1
[1] 2
second use:
[2] 1
[3] 2
```

Well, that’s surprising. The index didn’t start at zero for the second observable.

The second problem is that our operator will behave sensibly only when the observable it returns is subscribed to once.

Let’s look at what happens when multiple subscriptions are made to an observable composed from our `debug` operator:

```ts
import { range } from "rxjs";
import { debug } from "./debug";

const source = range(1, 2).pipe(debug());
console.log("first use:");
source.subscribe();
console.log("second use:");
source.subscribe();
```

The program’s output is:

```text
first use:
[0] 1
[1] 2
second use:
[2] 1
[3] 2
```

Again, it’s the same surprising behaviour: the index didn’t start at zero for the second subscription.

So how can these problems be fixed?

## The solution(s)

Both of the problems can be solved by storing the state on a per-subscription basis. And there are a number of ways this can be achieved.

The first is to use the `Observable` constructor to create the observable that our operator will return. If the `index` variable is moved into the function passed to the constructor, the index will be stored on a per-subscription basis. Like this:

```ts
import { MonoTypeOperatorFunction, Observable } from "rxjs";
import { tap } from "rxjs/operators";

export function debug<T>(): MonoTypeOperatorFunction<T> {
  return (source) =>
    new Observable<T>((subscriber) => {
      let index = -1;
      return source
        .pipe(tap((t) => console.log(`[${++index}]: ${t}`)))
        .subscribe(subscriber);
    });
}
```

The second way — which is my preferred way — of implementing per-subscription state is to use the `defer` observable creator. If the `index` variable is moved into the factory function passed to `defer`, it will be stored on a per-subscription basis. Like this:

```ts
import { defer, MonoTypeOperatorFunction } from "rxjs";
import { tap } from "rxjs/operators";

export function debug<T>(): MonoTypeOperatorFunction<T> {
  return (source) =>
    defer(() => {
      let index = -1;
      return source.pipe(tap((t) => console.log(`[${++index}]: ${t}`)));
    });
}
```

Another — more complicated — way of implementing per-subscription state is to use the `scan` operator. `scan` maintains its own per-subscription state which intialised with the `seed` and made available via the `accumulator`. The index can be stored within `scan` like this:

```ts
import { MonoTypeOperatorFunction } from "rxjs";
import { map, scan } from "rxjs/operators";

export function debug<T>(): MonoTypeOperatorFunction<T> {
  return (source) =>
    source.pipe(
      scan<T, [T, number]>(([, index], t) => [t, index + 1], [undefined!, -1]),
      map(([t, index]) => (console.log(`[${index}]: ${t}`), t))
    );
}
```

If the programs used to demonstrate the problems with the naive implementation are run using any of the implementations in which per-subscription state is maintained, the following output will be produced:

```text
first use:
[0] 1
[1] 2
second use:
[0] 1
[1] 2
```

Which is just what you’d expect: no surprises.
