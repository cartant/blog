---
title: "RxJS: Typing zipWith"
description: A look at how the TypeScript signature for zipWith works
date: "2020-02-10T18:36:00+1000"
categories: []
keywords: []
cardImage: "./title-card.jpeg"
---

![Coloured facade](title.jpeg "Photo by Ricardo Gomez Angel on Unsplash")

Much of what we are doing with RxJS, for version 7, is focused on improving the package's types.

RxJS version 6 was released in April 2018 and, at that time, the current version of TypeScript was 2.8. With the bump to version 7, we can update this minimum-supported version of TypeScript and can take advantage of some of TypeScript's more recently-added features to simplify the package's types whilst making them more accurate and more flexible.

This article, we'll look at the types that have been declared for a new operator: `zipWith`. In particular, we'll look at how they work and at the TypeScript features upon which they rely.

## What's zipWith?

`zipWith` isn't really a new operator; it's the deprecated `zip` operator renamed.

When RxJS version 6 was released, there were a handful of operators that had the same name as observable creators: `concat`; `combineLatest`; `merge`; `race`; and `zip`.

Prior to the introduction of `pipe`, these operators having the same name wasn't a problem, as operators were attached to `Observable.prototype`. However, when pipeable operators were introduced, the operators became static functions that had the same name as the creators:

```ts
import { zip } from "rxjs"; // the creator
import { zip } from "rxjs/operators"; // the operator
```

Operators are just functions that take an observable and return an observable, so the operators were deprecated and the recommendation was to use the creators instead, like this:

```ts
const answers = of(42).pipe((source) => zip(source, of(54)));
```

Anyway, the above-mentioned operators are being renamed: a `With` suffix is being added — yielding operator names similar to those in [RxJava](http://reactivex.io/RxJava/3.x/javadoc/io/reactivex/rxjava3/core/Observable.html#zipWith-io.reactivex.rxjava3.core.ObservableSource-io.reactivex.rxjava3.functions.BiFunction-) — and they can be used like this:

```ts
const answers = of(42).pipe(zipWith(of(54)));
```

`zipWith` isn't really new and it's not especially interesting, but the types in its declaration are representative of the changes that are happening in RxJS version 7, so let's take a look at those.

## Some background on RxJS's types

Let's look at two areas in which the package's types were problematic:

- union return types; and
- functions that take an arbitrary number of arguments.

Let's look at union return types first.

## Union return types

We'll use this as our example:

```ts
const result = of(Math.random()).pipe(
  concatMap((value) =>
    value < 0.05
      ? of("Yikes, rolled a 1!")
      : value < 0.95
      ? of(Math.floor(value * 20) + 1)
      : of("Yay, rolled a 20!")
  )
);
```

It might look a little contrived, but situations often arise in NgRx and `redux-observable` in which observable streams of different actions — i.e. different types — are returned from a projection function.

In RxJS version 6.3, the snippet would not have compiled and the following error would have been effected:

```text
Type 'Observable<number> | Observable<string>' is not
assignable to type 'ObservableInput<number>'.
```

The problem is that the projection function passed to `concatMap` returns either an `Observable<number>` or an `Observable<string>` — depending upon the roll — and it's not possible (or safe) for TypeScript to determine that `Observable<number | string>` is what should be inferred. Well, not without some help.

In January 2019, Google's [Alex Rickabaugh](https://github.com/alxhub) — he's on the Angular team — came up with a solution: the `ObservedValueOf` type. And it looks like this:

<!-- prettier-ignore -->
```ts
type ObservedValueOf<O> = O extends ObservableInput<infer T>
  ? T
  : never;
```

It extracts the value type from an `ObservableInput`. And it can be used like this:

```ts
function concatMap<T, O extends ObservableInput<any>>(
  project: (value: T, index: number) => O
): OperatorFunction<T, ObservedValueOf<O>>;
```

Using `ObservedValueOf` in the `concatMap` signature solves the problem and in our snippet, `result` is inferred to be `Observable<number | string>`. How it manages to solve the problem isn't immediately obvious, so let's take a closer look.

The `O extends ObservableInput<any>` type parameter imposes a constraint that the projection function's return value must be an `ObservableInput`. `Observable<number>` satisfies this constraint, as an `Observable` is a valid `ObservableInput`.

`Observable<number> | Observable<string>` also satisfies the constraint as both types within the union are `ObservableInput`s. That means that `O` is inferred to be `Observable<number> | Observable<string>` and that's what's passed to `ObservedValueOf`.

TypeScript's conditional types have an interesting property: they are [distributive](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html#distributive-conditional-types). That is, the "conditional types are automatically distributed over union types during instantiation."

That means that:

```ts
ObservedValueOf<Observable<number> | Observable<string>>
```

is effectively resolved as:

```ts
ObservedValueOf<Observable<number>> | ObservedValueOf<Observable<string>>
```

which is:

```ts
number | string;
```

And that's how `ObservedValueOf` solves the problem of projection functions returning union types.

`ObservedValueOf` was introduced in RxJS 6.4.0 and, unfortunately, it broke a whole bunch of Angular projects. The minimum-supported TypeScript version for RxJS version 6 was always considered to be 2.8 — as that was the latest TypeScript version when RxJS version 6 was released. However, Angular 6.1.0 [depended upon RxJS `^6.0.0` and TypeScript `~2.7.2`](https://github.com/angular/angular-cli/blob/52edee5c28c5013600f2f795c5f7b15b5c09935d/packages/schematics/angular/utility/latest-versions.ts#L12-L14) and that version of TypeScript did not support conditional types — which were introduced in version 2.8.

Because of this breakage, there have not been many subsequent changes to the types in RxJS version 6. Improvements to the package's types are constrained by the minimum-supported TypeScript version of 2.8 and by not being able to make changes to type parameters.

In RxJS version 7, the minimum-supported version of TypeScript will be bumped and numerous type parameters will be repurposed. We'll look at these repurposed type parameters next, when we see how another problem was solved: passing arbitrary numbers of arguments to functions.

## Arbitrary numbers of arguments

In September 2019, [Ben Lesh](https://twitter.com/BenLesh) added a type to support the passing of an arbitrary number of arguments to functions like `concat`. Like `ObservedValueOf` it's a conditional type and it looks like this:

<!-- prettier-ignore -->
```ts
type ObservedValuesFromArray<A> = A extends Array<ObservableInput<infer T>>
  ? T
  : never;
```

Prior to the introduction of `ObservedValuesFromArray`, functions like `concat` had to have [numerous overload signatures](https://github.com/ReactiveX/rxjs/blob/b8cb3a98d87d1db278b700ec35669f31e42bc258/src/internal/observable/concat.ts#L20-L31) — a signature for one argument, another for two arguments, etc. There were a limited number of these signatures — typically six — and if the number of arguments passed exceeded the limit, the type information was lost.

With `ObservedValuesFromArray`, only a single signature is needed and it's type-safe regardless of the number of arguments that are passed. Used with `concat` looks like this:

```ts
function concat<A extends ObservableInput<any>[]>(
  ...observables: A
): Observable<ObservedValuesFromArray<A>>;
```

It uses TypeScript's support for [rest elements in tuple types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-0.html#rest-elements-in-tuple-types) and when called like this:

```ts
const answers = concat(of(42), of("fifty-four"));
```

`A` will be inferred to be `[Observable<number>, Observable<string>]`.

The `[Observable<number>, Observable<string>]` tuple extends the array type `Array<number | string>`, so `ObservedValuesFromArray` will infer `T` to be `number | string`. And that means that `answers` will be inferred to be `Observable<number | string>`.

This is a huge improvement for the package's types. The elimination of overload signatures and the correct type inference — regardless of the number of passed arguments — are significant wins. However, the change is a breaking one, as the type parameter is repurposed. Let's take a closer look at the repurposing.

Prior to the removal of the overload signatures, the single-argument signature looked like this:

```ts
function concat<T>(o: ObservableInput<T>): Observable<T>;
```

So code like this would have been valid (weird and definitely not recommended, but valid nonetheless):

```ts
const result = concat<number>([]);
```

With the removal of the overload signatures, this will break — as the type parameter refers to the arguments' tuple type and not the observable's element type.

## Typing zip

The types for `zip` are similar to those used for `concat` but there is a difference: `zip` emits an array containing values received from its source observables. So the inference of an union type from a tuple type is not what's needed. What's needed is a [mapped-tuple type](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-1.html#mapped-types-on-tuples-and-arrays).

Taking this into account, `ObservedValuesFromArray` has been renamed to `ObservedValueUnionFromArray`:

<!-- prettier-ignore -->
```ts
type ObservedValueUnionFromArray<A> =
  A extends Array<ObservableInput<infer T>>
    ? T
    : never;
```

And another type — that infers a tuple — has been added:

<!-- prettier-ignore -->
```ts
type ObservedValueTupleFromArray<A> =
  A extends Array<ObservableInput<any>>
    ? { [K in keyof A]: ObservedValueOf<A[K]> }
    : never;
```

This looks complicated, but what it does is reasonably straightforward: if `A` is a tuple of `ObservableInput`s, it maps `A` to a tuple in which each element has the type of the `ObservableInput`'s value — using Alex's type.

Let's look at an example:

<!-- prettier-ignore -->
```ts
type Example = ObservedValueTupleFromArray<
  [Observable<number>, Observable<string>]
>;
```

Here, `Example` will be inferred to be `[number, string]`.

With the mapped-tuple type, `zip` is similar to `concat` and no longer requires overload signatures and correctly infers the types for an arbitrary number of arguments:

```ts
function zip<A extends ObservableInput<any>[]>(
  ...observables: A
): Observable<ObservedValueTupleFromArray<A>>;
```

## Typing zipWith

The typing for `zipWith` is similar to that for `zip`, but there is a small difference: not all of the elements in the zipped array are present in the arguments' tuple. The initial element in the zipped array will be the value received from the source observable to which the `zipWith` operator is applied.

That is, when it's used like this:

```ts
const answers = of(42).pipe(zipWith(of("fifty-four")));
```

the type of `answers` needs to be inferred as `[number, string]`, but `[string]` is what's inferred when `ObservedValueTupleFromArray` is applied to the arguments tuple.

To get the type that we need, `number` has to be be added to the head of the tuple.

We can do that using another type:

<!-- prettier-ignore -->
```ts
type Cons<T, A extends any[]> =
  ((arg: T, ...rest: A) => any) extends ((...args: infer U) => any)
    ? U
    : never;
```

This type is a little tricky. It's a conditional type that uses a relationship between two intermediate function types to infer a tuple type that adds `T` at the head of `A`. It works by forming an intermediate function that takes an argument of type `T` followed by arguments for each of the types in `A`. `U` is then inferred from the arguments of the intermediate function, effectively adding `T` to the head of `A`.

We can use `Cons` to add `T` at the tuple's head, like this:

```ts
function zipWith<T, A extends ObservableInput<any>[]>(
  ...observables: A
): OperatorFunction<T, Cons<T, ObservedValueTupleFromArray<A>>>;
```

And with the `zipWith` signature declared like this, `answers` — in the above snippet — is inferred to be `Observable<[number, string]>`.

## What's still to do?

For version 7, the core team will be taking the types that we've looked here and will be applying them to other functions in the package. The goal is to get as much of the API as possible supporting the inference of correct types when an arbitrary number of arguments are passed. And to get rid of all those overload signatures.
