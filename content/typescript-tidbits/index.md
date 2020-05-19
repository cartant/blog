---
title: TypeScript Tidbits
description: What building ts-action taught me about TypeScript
date: "2018-02-01T22:56:42.486Z"
categories: []
keywords: []
ckTags: ["1464980"]
cardImage: "./title.jpeg"
slug: /@cartant/typescript-tidbits-b89a022a6b19
---

![Macarons](title.jpeg "Photo by Tatiana Lapina on Unsplash")

This article is about TypeScript. It’s not specifically about Angular, NgRx, React or redux-observable; it’s about what I learned — the hard way — when I built [`ts-action`](https://github.com/cartant/ts-action).

## Literal types: widening and narrowing

TypeScript has the concept of string, numeric, boolean and enum literal types. Variables having a literal type may only be assigned the specified literal value. For example, this variable has a literal type:

```ts
let foo: "FOO";
```

And can only be assigned a value of `"FOO"`:

```ts
foo = "FOO"; // OK
foo = "BAR"; // Type '"BAR"' is not assignable to type '"FOO"'.
```

When a literal or a variable that has a literal type is assigned to a constant, a literal type is inferred — the inference is safe, as the constant’s value cannot be changed. Here, `foo` will be inferred to be of type `"FOO"`:

```ts
const foo = "FOO";
```

When a constant or variable with a literal type is assigned to a variable, the literal type cannot be inferred, as the variable’s value could be changed after the assignment. In these situations, the inference is widened:

```ts
const foo = "FOO"; // Inferred to be "FOO".
let val = foo; // Inferred to be string.
```

A literal type is also inferred when a constant or variable with a literal type is assigned to a read-only property. The inference is widened for properties that are not read-only:

```ts
const foo = "FOO";
class Foo {
  readonly type = foo; // Inferred to be "FOO".
  val = foo; // Inferred to be string.
}
```

`ts-action` relies upon the inference of literal types for read-only properties in its implementation of user-defined type guards for actions. Consider this snippet:

```ts
class Foo {
  readonly type = "FOO"
}

class Bar {
  readonly type = "BAR";
}

let action: Foo | Bar;
...
switch (action.type) {
case "FOO":
  // action is narrowed to be Foo.
  return ...;
case "BAR":
  // action is narrowed to be Bar.
  return ...;
}
```

In the snippet, the read-only `type` properties are inferred to be literal types: `"FOO"` and `"BAR"`. Because of the literal types, when `action.type` is used in the `switch` statement, TypeScript will narrow the type of `action` within each `case`. That is, because the `action` variable is either `Foo` or `Bar`, if its `type` property is `"FOO"`, it’s narrowed to an instance of `Foo` — allowing type-safe access to the action’s properties.

The narrowing only occurs because of the literal types. If the `type` properties are specified as strings, TypeScript does not narrow the `action` within each `case` — it will remain `Foo | Bar`.

## Literal-types as parameters

When used with generic functions, literal types behave as you’d expect. Here, `foo` will be inferred to be of type `"FOO"` as that is the type of the `"FOO"` literal:

```ts
function identity<T>(val: T): T {
  return val;
}
const foo = identity("FOO");
```

It’s possible to constrain `T` so that it can only be a `string` or a string-literal type, like this:

```ts
function identity<T extends string>(val: T): T {
  return val;
}
const foo = identity("FOO");
```

Again, `foo` will be inferred to be of type `"FOO"`. The constraint just prevents non-string values from being passed to the function.

Sometimes it’s useful to pass functions an options object containing a number of properties and it’s here that literal types get a little strange. Have a look at this snippet:

```ts
function identity<T>(options: { val: T }): T {
  return options.val;
}
const foo = identity({ val: "FOO" });
```

You might think that `foo` will still be inferred to be of type `"FOO"`, but it won’t be; it’ll be inferred to be a `string`. Using `readonly val` won’t help, either; it’ll still be inferred to be a `string`.

Interestingly, if `T` is constrained to extend `string`, the type will be inferred as you might expect (or hope):

```ts
function identity<T extends string>(options: { val: T }): T {
  return options.val;
}
const foo = identity({ val: "FOO" });
```

In the above snippet, `foo` will be inferred to be of type `"FOO"`.

## Mixin classes

TypeScript 2.2 introduced [mixin classes](https://github.com/Microsoft/TypeScript/wiki/What%27s-new-in-TypeScript#support-for-mix-in-classes). Mixin classes rely upon some complicated TypeScript features — the intersection of constructor signatures — but they are conceptually straightforward.

A mixin class extends a base class — about which the mixin knows nothing — but instances of the mixin class are constructed using the base class’s constructor’s signature. Let’s look at how they work.

A class constructor can be represented using this type:

```ts
type Ctor<T> = new (...args: any\[\]) => T;
```

To create a mixin class, a generic function that has a type parameter that extends `Ctor<{}>` is used. This mixin just adds a `mixed` property:

```ts
function mix<C extends Ctor<{}>>(Base: C) {
  return class extends Base {
    mixed = true;
    constructor(...args: any\[\]) {
      super(...args);
    }
  };
}
```

Note that the constructor used for the mixin class takes its arguments using the rest syntax and calls `super` using the spread syntax. TypeScript recognises this arrangement as a mixin class. And when an instance of the mixin class is created, the constructor arguments need to be those accepted by the `Base` class’s constructor. Let’s look at an example:

```ts
class Foo {
  constructor(public foo: string) {}
}
const MixedFoo = mix(Foo);
const mixed = new MixedFoo("mixed");
console.log(mixed); // { foo: "mixed", mixed: true }
```

Here, although the `mix` function known nothing about `Foo`, the returned mixin class’s constructor requires the same arguments as `Foo`’s constructor.

`ts-action` uses mixin classes in its `action` function. The `action` function takes a base class that represents action’s data and mixes in the literal-type properties necessary to represent the action’s type. Using a mixin the constructors in the created actions can accept either payloads, props or named parameters.

## Inferring a function’s return type

An interesting feature of mixin classes is that the functions that create them return constructors. And obtaining a constructor’s return type from the constructor is tricky.

One way of obtaining is to write a generic function with a signature takes a parameterised constructor, like this:

```ts
function returns<T>(C: Ctor<T>): T {
  return undefined!;
}
```

The function returns `undefined`, because the return _value_ isn’t going to be used — only the return _type_ will be used. The function can be used like this:

```ts
const Returned = false || returns(MixedFoo);
let r: typeof Returned;
```

In the snippet, `Returned` will be inferred to be of the type returned by the `MixedFoo` constructor. Note that `false` can be included in the expression to prevent the `returns` function from being called: TypeScript will infer the return type, but at runtime the function will not be called.

`ts-action` uses this mechanism in its `union` function — for constructing the union types that TypeScript requires for its type guards.

## Union and intersection types

Something that had always confused me with union and intersection types was that a union type contained the intersection of the types’ properties and an intersection type contained the union of the properties. It turned out I was just thinking about it the wrong way. As [Ryan Cavanaugh](https://stackoverflow.com/a/38857724/6680611) explains it:

> Here’s another way to think about it. Consider four sets: Red things, blue things, big things, and small things.
>
> If you **intersect** the set of all red things and all small things, you end up with the **union** of the properties — everything in the set has both the red property and the small property.
>
> But if you took the **union** of red small things and blue small things, only the smallness property is universal in the resulting set. **Intersecting** “red small” with “blue small” produces “small”.
>
> In other words, taking the union of the domain of values produces an intersected set of properties, and vice versa.

It all makes sense, now.

And that’s it; those are the TypeScript features and quirks that I got to know when building `ts-action`. Hopefully, the few minutes you’ve spent reading about them here has been as informative as the … well, let’s just say it took me a little longer than a few minutes understand all of this.
