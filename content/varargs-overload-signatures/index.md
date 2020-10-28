---
title: "TypeScript: Varargs Overload Signatures"
description: "The order of some signatures should not matter, but it does"
date: "2020-10-28T11:16:00+1000"
categories: []
keywords: []
ckTags: []
cardImage: "./title-card.jpeg"
---

![Wrenches](title.jpeg "Photo by Matt Artz on Unsplash")

I thought I'd write up a short post to explain a TypeScript problem that took up far too much of my time, yesterday.

The **TL;DR** is that if you have overload signatures for a function that takes a variable number of (initial) arguments, the ordering of the signatures matters in a way that is not obvious. And the 'simplest' signature should be placed first.

The types I was working with were in RxJS, but to keep things general let's use an example that combines a bunch of arrays in some way, like this:

```ts
const result = combine(["a", "b"], [1, 2]);
```

The `combine` function will take elements from each of the input arrays that are passed — the caller can pass as many as is necessary — and will return an array containing the combined elements. (How they are combined doesn't matter; we're only interested in the types.)

`combine` could be typed [like this](https://www.typescriptlang.org/play?#code/C4TwDgpgBAkgdmArsAzgHgKIBsIFsJyoB8UAvFAN4BQUUA2tngcFAJZxQDWEIA9gGZRG+QigC6ALigAnCAEMAJrzhYQQnCNQMNzMXTEBuKgF8jVBRADGWObKj9EcS8FbKol3rgBG7CJh2iUBAAHsAECigy8koqao6ccLwA7nD6RAAUNFAAdLnsSKhSdLnZ8AXowswoRGJUAJRSlaJ0cIjeENJ6hlRUHnAoLLIoiFgs5B7evul0AERyMwA0UDNeM2JLdACMSwBMYnUGQA):

```ts
type Inputs<Elements> = {
  [Element in keyof Elements]: readonly Elements[Element][];
};

declare function combine<Elements extends readonly unknown[]>(
  ...inputs: [...Inputs<Elements>]
): Elements[number][];
```

Without going into too much detail, `Inputs` is mapped type that maps a tuple of element types to a tuple of read-only arrays — for the parameters — like this:

```ts
type Mapped = Inputs<[string, number]>;
// [readonly string[], readonly number[]];
```

And the element type of the returned array is obtained — from the `Elements` type parameter — using this mechanism:

```ts
type Zeroth = [string, number][0]; // string
type All = [string, number][number]; // string | number
```

With the signature declared like this, the return value's type is inferred correctly:

```ts
const result = combine(["a", "b"], [1, 2]);
// (string | number)[]
```

Let's say the `combine` function can also be passed a count to specify how many elements at a time should be taken from each array. We can add an overload signature [like this](https://www.typescriptlang.org/play?#code/C4TwDgpgBAkgdmArsAzgHgKIBsIFsJyoB8UAvFAN4BQUUA2tngcFAJZxQDWEIA9gGZRG+QigC6ALigAnCAEMAJrzhYQQnCNQMNzMXTEBuKgF8jVBRADGWObKj9EcS8FbKol3rgBG7CJh2iUBAAHsAECigy8koqao6ccLwA7nD6RAAUNFAAdLnsSKgAgnAKAMK8jsBSdLnZ8AXowswoRAA0UHCI3hDSYlQAlFJNonSd3b36RhbWttAOTi5uHt6+-kyBIWElkbKKyqpQ8YkpaZm0tfnIKNW19Vdrmi19g+rrWmNePXqGVFQecCgWLIUIgsCxyMsfHAIOk6AAiORw9pwrxwsTtOgARnaACZ0VAcf0DEA):

```ts
declare function combine<Elements extends readonly unknown[]>(
  ...inputsAndCount: [...Inputs<Elements>, number]
): Elements[number][];
declare function combine<Elements extends readonly unknown[]>(
  ...inputs: [...Inputs<Elements>]
): Elements[number][];
```

With these signatures, the return value's type is still inferred correctly:

```ts
const result = combine(["a", "b"], [1, 2], 2);
// (string | number)[]
```

Let's say we add [another overload signature](https://www.typescriptlang.org/play?#code/C4TwDgpgBAkgdmArsAzgHgKIBsIFsJyoB8UAvFAN4BQUUA2tngcFAJZxQDWEIA9gGZRG+QigC6ALigAnCAEMAJrzhYQQnCNQMNzMXTEBuKgF8jVBRADGWObKj9EcS8FbKol3rgBG7CJh2iUBAAHsAECigy8koqao6ccLwA7nD6RAAUNFAAdLnsSKgAgnAKAMK8jsDFCgAirNJSdLnZ8AXowswoRAA0UHCI3hDSvQBEOPzAI1AAPlAj0qwA5gAWk2JUAJRSHaJ0-YPSeobmVjZ2Dk4ubh7evv5MgSFhJZGyisqqUPGJKWmZtM18sgUNVypVGs1WsD7pour19l4hustuoHloEUj9EYLNZbNALs5XBwbj44H4dqggqFwq9oh84nAEslUmIMllAQhgRDclDUDDOkRkdsAuiBojDliqFQPHAUCxZChEFgWOQSb50nQRnIRqMvCMxL06ABGXoAJgNUFNowWK0mGwMQA) that allows the direction of the combination to be specified:

```ts
declare function combine<Elements extends readonly unknown[]>(
  ...inputsAndCountAndDir: [...Inputs<Elements>, number, "left" | "right"]
): Elements[number][];
declare function combine<Elements extends readonly unknown[]>(
  ...inputsAndCount: [...Inputs<Elements>, number]
): Elements[number][];
declare function combine<Elements extends readonly unknown[]>(
  ...inputs: [...Inputs<Elements>]
): Elements[number][];
```

With these signatures, the return value's type is still inferred correctly for this usage:

```ts
const result = combine(["a", "b"], [1, 2], 2, "right");
// (string | number)[]
```

However, there is a problem. If `combine` is passed only a single input, the return value's type is not inferred correctly:

```ts
const result = combine(["a", "b"]);
// unknown[]
```

This doesn't make much sense. The only overload signature that should be matched is the last one.

It turns out that the order of the overload signatures is significant. It shouldn't be, but it is.

If they are ordered [like this](https://www.typescriptlang.org/play?#code/C4TwDgpgBAkgdmArsAzgHgKIBsIFsJyoB8UAvFAN4BQUUA2tngcFAJZxQDWEIA9gGZRG+QigC6ALigAnCAEMAJrzhYQQnCNQMNzMXTEBuKgF8jVBRADGWObKj9EcS8FbKol3rgBG7CJh2iUBAAHsAECigy8koqao6ccLwA7nD6RAAUNFAAdLnsSKhSdLnZ8AXowswoRGJUAJRSlaJ0cIjeENJ6huZWNnYOTi5uHt6+-kyBIWFwEVGKyqpQ8YkpaZm0JfnIKACCMwDCvI7ARSVl2+Oa1QA0UK3tnfWNAVr3Xh1dRhbWttADzq4OCMfHA-E1UEFQuFIrJ5rElnAEslUmIMllNghtnsFIdjtiACKsaSnXLnVCXKpEW5vDq3ABEOH4wDpUAAPlA6dJWABzAAWzNqDXUE1ebXenX0Zg8cBQLFkKEQWBY5GBvnSdDpcjp9K8dLEdQMQA):

```ts
declare function combine<Elements extends readonly unknown[]>(
  ...inputs: [...Inputs<Elements>]
): Elements[number][];
declare function combine<Elements extends readonly unknown[]>(
  ...inputsAndCount: [...Inputs<Elements>, number]
): Elements[number][];
declare function combine<Elements extends readonly unknown[]>(
  ...inputsAndCountAndDir: [...Inputs<Elements>, number, "left" | "right"]
): Elements[number][];
```

the inference is correct:

```ts
const result = combine(["a", "b"]);
// string[]
```

So the take away from this is that if you have overloaded varargs signatures, they should be ordered so that the first signature is the signature without trailing parameters.
