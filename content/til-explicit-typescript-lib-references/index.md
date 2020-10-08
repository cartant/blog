---
title: "TIL: Explicit TypeScript lib References"
description: "How to ensure a package's depended-upon types are referenced"
date: "2020-10-08T22:32:00+1000"
categories: []
keywords: []
ckTags: ["1464980"]
cardImage: "./title-card.jpeg"
---

![Shoes with stripes](title.jpeg "Photo by Eddie Palmore on Unsplash")

## The problem

When a package uses a modern JavaScript feature, its types can behave strangely if the consuming application's TypeScript configuration isn't up-to-date with the feature.

For example, if the package uses an `ES2018` feature — like [async iterables](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of) — but the application's TypeScript configuration is `ES2015`, the behaviour can be surprising and difficult to understand.

This has happened with RxJS. In version 7, an async iterable can be used as an observable input — it can be passed to an observable creator, like [`from`](https://rxjs.dev/api/index/function/from) — so the [`ObservableInput`](https://github.com/ReactiveX/rxjs/blob/a331e11696531521ccf0ae62d97b0b8d1f26f28e/src/internal/types.ts#L79) type looks like this:

<!-- prettier-ignore -->
```ts
export type ObservableInput<T> =
  | SubscribableOrPromise<T>
  | ArrayLike<T>
  | Iterable<T>
  | AsyncIterableIterator<T>;
```

If the TypeScript configuration for the application that consumes RxJS does not have `lib` configured as `ES2018` or later, the `AsyncIterableIterator` type won't be referenced and the type inference will behave in an unexpected manner.

Let's take a look at this snippet:

```ts
const answers = from([42, 54]);
```

The observable input is an array of numbers, so the inferred type of `answers` should be `Observable<number>`, but it's not. If `lib` is configured as `ES2017` or earlier, the inferred type is `Observable<unknown>`.

And it's not at all clear that the unexpected inference is related to the TypeScript configuration — in particular, to the `lib` that's specified. 😬

## How does the configuration work?

There are two ways the [TypeScript `lib` compiler option](https://www.typescriptlang.org/tsconfig#lib) can be configured.

1. No `lib` option is specified. In which case, TypeScript uses a the `target` and adds references to the `DOM` and `ScriptHost` libraries.

   For example, if the `target` is `ES5`, the `lib` option will be `["ES5", "DOM", "ScriptHost"]`.

2. The `lib` option can be specified explicitly. In which case, the `target` option has no bearing on which libraries are referenced.

It's common for the `lib` option to be specified because:

- some developers (like me, before I wrote this post 😅) don't read all of the TypeScript documentation and don't understand how the `target` and `lib` options interact; and
- Node developers don't want the `DOM` types to be referenced.

That means it's quite likely that there will be some applications that won't have TypeScript configured for modern features used within packages upon which they depend.

## A solution

The problem can be avoided if the package is able to have some say in the TypeScript libraries that are referenced. [Ryan Cavanaugh](https://twitter.com/SeaRyanC) [pointed out](https://github.com/microsoft/TypeScript/issues/40462#issuecomment-689879308) that this can be done by adding a TypeScript [triple-slash directive](https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html#-reference-lib-) to specify a `lib` reference, like this:

```text
/// <reference lib="ESNext.AsyncIterable" />
```

The directive can be added to a `.ts` source file. When TypeScript compiles the source file, it will include the directive in the generated `.d.ts` file. Then, when the consuming application imports the `.d.ts` file, the directive will be processed and the specified library will be referenced.

The solution works. It resolves the problem mentioned above — with the directive, the type is correctly inferred to be `Observable<number>` — but it's not perfect. Ryan mentions that referencing `ESNext.AsyncIterable` also references a handful of other types — some of which might not have been otherwise referenced by the application developer's TypeScript configuration.

The solution is, however, much better than the alternative: countless developer hours wasted and an interminable number of issues opened. I mean, we will, of course, mention RxJS's `lib` requirement — a minimum of `ES2018` — in the documentation, but who's going to read that? Probably not me. 🙂
