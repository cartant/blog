---
title: "RxJS: Improving the Static pipe Function"
description: Adding type-safe support for generics
date: "2018-05-21T07:15:34.331Z"
categories: []
keywords: []
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-improving-the-static-pipe-function-81146fbb14b6
---

![Pencil sharpening](title.jpeg "Photo by Angelina Litvin on Unsplash")

My [previous article](/combining-operators/) looked at using the static `pipe` function to compose reusable combinations of operators.

Most of the time, the `pipe` function’s TypeScript overload signatures will infer the desired type for the returned function. However, sometimes it’s desirable to have a generic type inferred and the current overload signatures will not do that.

Let’s look at why you might want a generically-typed function to be returned and what needs to be changed for that to happen.

## Specifically-typed operators

The static `pipe` function can be used to compose a reusable, debouncing operator that can be used with the changes emitted by `input` elements:

```ts
import { pipe } from "rxjs";
import { debounceTime, distinctUntilChanged } from "rxjs/operators";

export const debounceInput = pipe(
  debounceTime<string>(400),
  distinctUntilChanged()
);
```

The static `pipe` function does not have a source observable, so there is nothing from which the first type parameter can be inferred.

If an explicit type parameter is not specified, it will be inferred as `{}` — an empty interface — because that’s how TypeScript behaves if a type parameter is not specified and cannot be inferred.

If a type parameter is not specified — and `{}` is inferred — applying the composed operator to a source observable will see the resultant observable inferred as `Observable<{}>` — losing the source observable’s type.

The type parameter is specified on the `debounceTime` operator — rather than on the `pipe` call itself — as the `pipe` call has more than one type parameter and specifying only the first will see the remaining type parameters inferred as `{}`. And specifying them all is too tedious.

A `string` type parameter means that the composed operator can only be used with observable sources that emit `string` values. Here, that’s unlikely to be a problem, as `input` elements will be emitting `string` values when they change.

However, sometimes you don’t want to restrict composed operators to observables of a specific type and that’s when you need `pipe` to return a generically-typed function.

## Generically-typed operators

To synchronise the rendering of changes with the browser, it’s common to use the `auditTime` operator with the `animationFrameScheduler`.

To avoid having to specify the arguments in each call, we could compose `auditFrame`, like this:

```ts
import { animationFrameScheduler } from "rxjs";
import { auditTime } from "rxjs/operators";

export const auditFrame = auditTime(0, animationFrameScheduler);
```

However, for the reasons outlined above, when `auditFrame` is applied to a source observable, the source observable’s type will be lost.

How can we compose a generically-typed `auditFrame`? That is, how can we compose an `auditFrame` function that will have a signature something like this?

```ts
import { Observable } from "rxjs";

declare const auditFrame: <T>(source: Observable<T>) => Observable<T>;
```

If the static `pipe` function were to return a generically-typed function, we could compose `auditFrame` like this:

```ts
import { animationFrameScheduler, pipe } from "rxjs";
import { auditTime } from "rxjs/operators";

export const auditFrame = pipe(auditTime(0, animationFrameScheduler));
```

And could then use `auditFrame` — in a type-safe way — with source observables of any type.

So how can we change `pipe` to return a generically-typed function?

## Overload signatures

There are numerous operators — like `auditTime`, `debounceTime`, and `distinctUntilChanged`— that don’t change emitted values and don’t change the source observable’s type.

It’s simple to add an overload signature that matches their use, as each operator accepts and returns an observable of the same type. Such an overload signature — that returns generically-typed function — would look something like this:

```ts
import { MonoTypeOperatorFunction, Observable, UnaryFunction } from "rxjs";

declare module "rxjs/internal/util/pipe" {
  export function pipe<T>(
    ...operators: MonoTypeOperatorFunction<T>[]
  ): <R extends T>(source: Observable<R>) => Observable<R>;
}
```

With the overload signature declared, whenever `pipe` is passed one or more operators that deal only with observables of type `T`, the returned function can be used with any observable of type `R` — where `R` extends `T`.

That means that the `auditFrame` composed using `pipe` can be used with source observables of any type, without the source observable’s type being lost. And it also means that `debounceInput` doesn’t have to have an explicit type parameter specified.

Interestingly, this particular overload signature needs to precede the others. That means you don’t have to wait for the signature to be incorporated into RxJS and can instead perform the declaration merging done in the snippet to add the overload signature to a current codebase — as merged signatures are placed first.

Or you can write your own, as I’ve [done here](https://github.com/cartant/rxjs-etc/blob/v7.2.1/source/genericPipe.ts#L8-L22).

---

_If you are wondering whether or not there is a PR for the additional overload signature, there isn’t, but I’ll be opening one once some type-checking issues with the RxJS test suite are resolved._
