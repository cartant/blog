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

`combine` could be typed [like this](https://www.typescriptlang.org/play?#code/C4TwDgpgBAkgdmArsAzgHgKIBsIFsJyoB8UAvFAN4BQUUA2gNIQhQCWcUA1swPYBmUbHgKoAugC4oAJwgBDACY84WFkPyEUjZqLqiA3FQC+BqvIgBjLLJlQ+iOOeCslUcz1wAjdhEw51qKAgAD2ACeRQoe044HgB3OF0iAAoaKAA6DPYkVEk6DLT4bPQ1ERQiUSoASkkSjTo4RE8IKR19Kio3OBRgaQgURCwe8jdPbyS6ACJZCYAaKAmPCdE5ugBGOYAmUUq9IA):

```ts
type Inputs<Elements> = {
  [Key in keyof Elements]: readonly Elements[Key][];
};

declare function combine<Elements extends unknown[]>(
  ...inputs: [...Inputs<Elements>]
): Elements[number][];
```

Without going into too much detail, `Inputs` is mapped type that maps a tuple of element types to a tuple of read-only arrays — for the input array parameters — like this:

```ts
type Mapped = Inputs<[string, number]>;
// [readonly string[], readonly number[]];
```

And the element type of the returned array is obtained — from the `Elements` type parameter — using this mechanism:

```ts{7-8}
type Element0 = [string, number][0];
// string

type Element1 = [string, number][1];
// number

type All = [string, number][number];
// string | number
```

With the signature declared like this, the return value's type is inferred correctly:

```ts
const result = combine(["a", "b"], [1, 2]);
// (string | number)[]
```

Let's say the `combine` function can also be passed a count to specify how many elements at a time should be taken from each input array. We can add an overload signature [like this](https://www.typescriptlang.org/play?#code/C4TwDgpgBAkgdmArsAzgHgKIBsIFsJyoB8UAvFAN4BQUUA2gNIQhQCWcUA1swPYBmUbHgKoAugC4oAJwgBDACY84WFkPyEUjZqLqiA3FQC+BqvIgBjLLJlQ+iOOeCslUcz1wAjdhEw51qKAgAD2ACeRQoe044HgB3OF0iAAoaKAA6DPYkVABBOHkAYR57YEk6DLT4bPQ1ERQiABooOERPCClRKgBKSVqNOha2jt0DM0traDsHJxc3T29fYQ1AkLCIqJj4xJTaCqzkFDKKqoPF-3rOnsE-OoHWj3adfSoqNzgUYGkIFEQsT-I5l44BAknQAESyMFNMEeMGiJp0ACMTQATPCoCiunogA):

```ts{1-3}
declare function combine<Elements extends unknown[]>(
  ...inputsAndCount: [...Inputs<Elements>, number]
): Elements[number][];
declare function combine<Elements extends unknown[]>(
  ...inputs: [...Inputs<Elements>]
): Elements[number][];
```

With these signatures, the return value's type is still inferred correctly:

```ts
const result = combine(["a", "b"], [1, 2], 2);
// (string | number)[]
```

Let's say we add [another overload signature](https://www.typescriptlang.org/play?#code/C4TwDgpgBAkgdmArsAzgHgKIBsIFsJyoB8UAvFAN4BQUUA2gNIQhQCWcUA1swPYBmUbHgKoAugC4oAJwgBDACY84WFkPyEUjZqLqiA3FQC+BqvIgBjLLJlQ+iOOeCslUcz1wAjdhEw51qKAgAD2ACeRQoe044HgB3OF0iAAoaKAA6DPYkVABBOHkAYR57YDz5ABFWKUk6DLT4bPQ1ERQiABooOERPCCkOgCIcPmB+qAAfKH6pVgBzAAsR0SoASklmjTounqkdfVMLKxs7BycXN09vX2ENQJCwiKiY+MSU2jqs5BQyopKauobPld-K0OlsPL0lqtBH4WptuuCdroDGZLNZoMdHM4OOcvHAfOsAsFQvkHnBonEEqJkql3ghPn8MgDUECWkRIWsYRswRCkVQqG44ChgNIIChEFhheQcd4knR+rJ+gMPP1RB06ABGDoAJlVUC1A2m8xGyz0QA) that allows the direction of the combination to be specified:

```ts{1-3}
declare function combine<Elements extends unknown[]>(
  ...inputsAndCountAndDir: [...Inputs<Elements>, number, "left" | "right"]
): Elements[number][];
declare function combine<Elements extends unknown[]>(
  ...inputsAndCount: [...Inputs<Elements>, number]
): Elements[number][];
declare function combine<Elements extends unknown[]>(
  ...inputs: [...Inputs<Elements>]
): Elements[number][];
```

With these signatures, the return value's type is inferred correctly for this usage:

```ts
const result = combine(["a", "b"], [1, 2], 2, "right");
// (string | number)[]
```

However, there is a problem. If `combine` is passed only a single input array, the return value's type is not inferred correctly:

```ts
const result = combine(["a", "b"]);
// unknown[]
```

This doesn't make much sense. The only overload signature that should be matched is the last one.

It turns out that the order of the overload signatures is significant. It shouldn't be, but it is.

If they are ordered [like this](https://www.typescriptlang.org/play?#code/C4TwDgpgBAkgdmArsAzgHgKIBsIFsJyoB8UAvFAN4BQUUA2gNIQhQCWcUA1swPYBmUbHgKoAugC4oAJwgBDACY84WFkPyEUjZqLqiA3FQC+BqvIgBjLLJlQ+iOOeCslUcz1wAjdhEw51qKAgAD2ACeRQoe044HgB3OF0iAAoaKAA6DPYkVEk6DLT4bPQ1ERQiUSoASkkSjTo4RE8IKR19UwsrGzsHJxc3T29fYQ1AkLCIqJj4xJTafKzkFABBOHkAYR57YFz8wsWh-zKAGigGppaqmr9S+saPZtaDM0traG7HZw5+rzgfWoDgqFVhM4NE4glRMlUvMEIsVutNoR4QARVhSHYZPaoA6lIgnM73KQnABEOD4wGJUAAPlBiVJWABzAAWFIq1UE1zqBIeuhMbjgKGA0ggKEQWCF5G+3iSdGJsmJJI8xNElT0QA):

```ts
declare function combine<Elements extends unknown[]>(
  ...inputs: [...Inputs<Elements>]
): Elements[number][];
declare function combine<Elements extends unknown[]>(
  ...inputsAndCount: [...Inputs<Elements>, number]
): Elements[number][];
declare function combine<Elements extends unknown[]>(
  ...inputsAndCountAndDir: [...Inputs<Elements>, number, "left" | "right"]
): Elements[number][];
```

the inference is correct:

```ts
const result = combine(["a", "b"]);
// string[]
```

So the take away from this is that if you have overloaded varargs signatures, they should be ordered so that the first signature is the signature without trailing parameters.
