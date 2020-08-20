---
title: "Catching Unknowns"
description: "Lint rules for type-safe catching"
date: "2020-07-14T10:41:00+1000"
categories: []
keywords: []
ckTags: ["1464979", "1464980"]
cardImage: "./title-card.jpeg"
---

![Parrot](title.jpeg "Photo by Ignacio Amenábar on Unsplash")

TypeScript version 4 is going to introduce a bunch of [new features](https://devblogs.microsoft.com/typescript/announcing-typescript-4-0-beta) and one of those will be allowing `unknown` [to be specified as the type for `catch`-clause variables](https://devblogs.microsoft.com/typescript/announcing-typescript-4-0-beta/#unknown-on-catch).

At the moment, `catch`-clause variables are implicitly typed as `any`, so — as far as the compiler is concerned — you can do whatever you want with them:

```ts
try {
  /* ... */
} catch (error) {
  console.error(error.message); // YOLO
}
```

This isn't changing in version 4 — `catch`-clause variables will still be implicitly typed as `any` — but it might change in the future:

> While the types of catch variables won’t change by default, we might consider a new `--strict` mode flag in the future so that users can opt in to this behavior.

In the meantime, `catch` clauses can be made type-safe by explicitly specifying `unknown` as the type:

<!-- prettier-ignore -->
```ts
try {
  /* ... */
} catch (error: unknown) {
  console.error(error.message); // error TS2571: Object is of type 'unknown'.
}
```

Here, with the `catch`-clause variable typed as `unknown`, a type guard is necessary to establish whether or not the caught value is actually an `Error` instance:

<!-- prettier-ignore -->
```ts
try {
  /* ... */
} catch (error: unknown) {
  if (error instanceof Error) {
    console.error(error.message); // It's an Error instance.
  } else {
    console.error("🤷‍♂️"); // Who knows?
  }
}
```

`unknown` and `any` are the only allowed types for `catch`-clause variables. There is no mechanism for constraining errors that could be thrown from within the `try` block, so specifying a type narrower than `unknown` or `any` would be unsafe.

To encourage type-safe `catch` clauses, a new rule is being added to the [TypeScript ESLint rules](https://github.com/typescript-eslint/typescript-eslint): [`no-implicit-any-catch`](https://github.com/typescript-eslint/typescript-eslint/pull/2202).

It will effect lint failures for implicit-`any` `catch` clauses and will suggest using an explicit `unknown`. Enabling it will highlight any unsafe assumptions that are being made within your `catch` blocks.

However, the problem of errors being typed as `any` isn't limited to `catch` blocks. There are two other situations where potentially unsafe code could be lurking.

With a `Promise`, the rejection callback's `error` parameter is typed as `any`:

```ts
somePromise.catch((error) => {
  console.error(error.message); // YOLO
});
```

And with an RxJS `Observable`, the `catchError` operator's function receives an `error` parameter that is typed as `any`:

```ts
someObservable.pipe(
  catchError((error) => {
    console.error(error.message); // YOLO
  })
);
```

In both of these situations, unsafe assumptions would effect errors if the `error` parameters were typed as `unknown`, like this:

```ts
somePromise.catch((error: unknown) => {
  console.error(error.message); // error TS2571: Object is of type 'unknown'.
});
```

I've added a [`no-implicit-any-catch` rule](https://github.com/cartant/eslint-plugin-etc/blob/ecb45035b24c26995432fee4864b5ff88b9a7b40/source/rules/no-implicit-any-catch.ts) to `eslint-plugin-etc` to ensure that `Promise` rejections have `error` parameters typed as `unknown` and a [`no-implicit-any-catch` rule](https://github.com/cartant/eslint-plugin-rxjs/blob/d388160a170ae5b1bae19f192c95e431436dfe40/source/rules/no-implicit-any-catch.ts) to `eslint-plugin-rxjs` to do the same for `catchError` operators.

The rules have suggestions and fixers for automatically updating code to ensure `unknown` is used as the type. The rules will also ensure that an `error` parameter's type is not unsafely narrowed — as with `catch` clauses, there is no constraint on what could be thrown as an error, so specifying a narrower type would be unsafe.

I've also added TSLint rules for devs who've not yet made the switch to ESLint. There is an `rxjs-no-implicit-any-catch` rule in [`rxjs-tslint-rules`](https://github.com/cartant/rxjs-tslint-rules) that supports `catchError` and there is a `no-implicit-any-catch` rule in [`tslint-etc`](https://github.com/cartant/tslint-etc) that supports both `Promise` rejections and `catch` clauses.

My thanks go to [Felix Becker](https://twitter.com/felixfbecker) for suggesting the rules and their behaviour.
