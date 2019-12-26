---
title: "RxJS: How to Use Type Guards with Observables"
description: Using type guards for run-time validation
date: "2017-09-11T05:19:34.415Z"
categories: []
keywords: []
slug: /@cartant/rxjs-how-to-use-type-guards-with-observables-11cc4d4f380f
---

![Chain-link fence](title.jpeg "Photo by Tanner Van Dera on Unsplash")

Since [version 1.6](https://github.com/Microsoft/TypeScript/wiki/What’s-new-in-TypeScript#typescript-16), TypeScript has supported user-defined type guards.

When composing an observable, a type guard can be used to ensure that the correct type is inferred at compile time and that the received value is validated at run time. With run-time validation, problems can be caught early and descriptive errors can be thrown from locations close to where values are received — these are easier to diagnose than cannot-read-property-of-`undefined` errors that are thrown from seemingly unrelated locations.

## What is a type guard?

A user-defined type guard is a function that performs a run-time check to evaluate its returned type predicate.

Let’s look at an example that uses the following interface:

```ts
interface Person {
  name: string;
  age: number;
}
```

A basic type guard that can be used to determine whether or not a value is compatible with the `Person` interface looks something like this:

```ts
function isPerson(value: any): value is Person {
  return (
    value && typeof value.name === "string" && typeof value.age === "number"
  );
}
```

Of particular interest is the function signature’s return type: `value is Person`. This is the type predicate and it’s what makes the function a type guard.

The guard’s signature tells TypeScript that the function will return a value that represents the evaluation of the predicate, so our user-defined type guard can be used like this:

```ts
function toString(value: Person | string): string {
  if (isPerson(value)) {
    return `name = ${value.name}; age = ${value.age}`;
  } else {
    return value;
  }
}
```

Recognising `isPerson` as a type guard, TypeScript knows that if it returns a truthy result, `value` must be a `Person` and the `name` and `age` properties can therefore be used within the `if` block statement.

TypeScript also knows that if `isPerson` returns a falsy result, `value` must be a `string` — as that is the only alternative.

## How do we use a type guard with an observable?

With RxJS, there is often more than one way of implementing behaviour and that’s the case with our type guard. Let’s look at what it is we are trying to do.

A type guard evaluates a type predicate for a specified value and returns a boolean result. We want to use a type guard so that the correct type is inferred at compile time. Although we are not changing the value in any way, this is conceptually similar to the `map` operator: before using the guard, the observable will have a general type; and after using the guard, it will have a more specific type.

Let’s create a general function that will accept a type guard and will return a projection function that can be passed to the `map` operator:

```ts
function guard<T, R extends T>(
  r: (value: T) => value is R,
  message?: string
): (value: T) => R {
  return value => {
    if (r(value)) {
      return value;
    }
    throw new Error(message || "Guard rejection.");
  };
}
```

The `guard` function takes a type guard — `r` — and an optional rejection message. The returned ‘projection’ function doesn’t project values; it just returns values that pass the type guard and throws errors for values that do not.

When calling the `guard` function, its type parameters — `T` and `R` — can be inferred from the type guard, so they don’t need to be specified explicitly.

Let’s see how our `guard` function could be used with Angular’s [`HttpClient`](https://angular.io/guide/http). When the client’s `get` method is called like this:

```ts
const person = http.get(`/people/${id}`);
```

The type of `person` will be inferred as `Observable<Object>` — the return type of the `get` method. `Object` is not a particularly useful type, so the recommended way of calling `get` involves specifying a type parameter, like this:

```ts
const person = http.get<Person>(`/people/${id}`);
```

When called this way, the type of `person` will be inferred as `Observable<Person>`. This is a compile-time type assertion. It informs TypeScript that the response’s content will have a shape compatible with the `Person` interface. However, at run time, the response’s content could be anything.

When the `guard` function is used with the `map` operator, like this:

```ts
const person = http.get(`/people/${id}`).map(guard(isPerson));
```

The type of `person` will be inferred as `Observable<Person>` and a run-time check will be performed on the response’s content, effecting an easy-to-diagnose error if the content fails the type guard.

RxJS is nothing if not flexible and there are other ways of performing the compile-time type assertions and the run-time validations that we’ve looked at in this article. However, if you are already using interfaces and type guards, a general `guard` function makes it easy to add assertions and validations to observables that receive content from external sources. It’s an approach that I’ve found to be effective when using Angular’s `HttpClient`, [AngularFire2](https://github.com/angular/angularfire2)’s [`list`](https://github.com/angular/angularfire2/blob/master/docs/3-retrieving-data-as-lists.md) and [`object`](https://github.com/angular/angularfire2/blob/master/docs/2-retrieving-data-as-objects.md) observables, and RxJS’s [`ajax`](https://github.com/ReactiveX/rxjs/blob/5.4.3/src/observable/dom/AjaxObservable.ts#L101-L126) observable.
