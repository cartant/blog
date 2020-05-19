---
title: "RxJS: How to Use Lettable Operators and Promises"
description: A look at lettable operators and the toPromise function
date: "2017-10-03T10:45:58.165Z"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-how-to-use-lettable-operators-and-promises-2e717313bf76
---

![Glove on a picket](title.jpeg "Photo by Gary Bendig on Unsplash")

_With the release of_ [_RxJS 5.5.0-beta.5_](https://github.com/ReactiveX/rxjs/blob/5.5.0-beta.5/CHANGELOG.md#550-beta5-2017-10-06)_,_ `toPromise` _is now attached to_ `Observable.prototype`_, rendering this look at its ‘lettable’ variant a historical curiosity_.

---

Converting observables to promises is an antipattern. Unless you are integrating observables with a promise-based API, there is no reason to convert an observable into a promise.

If you are integrating with such an API, in his article [_RxJS Observable interop with Promises and Async-Await_](https://medium.com/@benlesh/rxjs-observable-interop-with-promises-and-async-await-bebb05306875), Ben Lesh shows how it can be done. However, that article uses the prototype-patched `toPromise` method. Let’s have a look at the how the ‘lettable’ `toPromise` function — introduced in the RxJS 5.5.0 beta — could be used.

With `toPromise`, ‘lettable’ is something of a misnomer, as it cannot be used with the `let` operator. [Lettable operators](/understanding-lettable-operators/) are higher-order functions that return functions that receive and return observables. The `toPromise` higher-order function returns a function that receives an observable, but it returns a promise — so it’s not lettable.

To convert an observable to a promise, `toPromise` needs to be called and the observable needs to be passed to the function it returns, like this:

```ts
import { range } from "rxjs/observable/range";
import { map, filter, scan, toArray, toPromise } from "rxjs/operators";

const value$ = range(0, 10).pipe(
  filter((x) => x % 2 === 0),
  map((x) => x + x),
  scan((acc, x) => acc + x, 0),
  toArray()
);

const value = toPromise()(value$);
value.then((x) => console.log(x)); // [0, 4, 12, 24, 40]
```

The `toPromise` call cannot be placed within the call to `pipe`, as `pipe` will only accept functions that receive and return observables. Instead, it must be called separately.

Usually, `toPromise` would be called without an argument. However, if a particular promise implementation is to be used, a constructor can be passed. For example, a Bluebird promise could used, like this:

```ts
import { Promise as Bluebird } from "bluebird";
import { of } from "rxjs/observable/of";
import { toPromise } from "rxjs/operators";

const value$ = of("foo");
const value = toPromise(Bluebird as any)(value$);
value.then((x) => console.log(x)); // "foo"
```

An `any` assertion is required because Bluebird does not have a `Symbol.species` property, making TypeScript deem it incompatible with `typeof Promise.`

If the [TC39 pipeline operator proposal](/pipelining-lettable-operators/) is eventually accepted into the ECMAScript standard, there will be an alternative, arguably-more-natural syntax, that looks like this:

```ts
import { range } from "rxjs/observable/range";
import { map, filter, scan, toArray } from "rxjs/operators";

const value = range(0, 10)
  |> filter(x => x % 2 === 0)
  |> map(x => x + x)
  |> scan((acc, x) => acc + x, 0)
  |> toArray()
  |> toPromise();

value.then(x => console.log(x)); // [0, 4, 12, 24, 40]
```
