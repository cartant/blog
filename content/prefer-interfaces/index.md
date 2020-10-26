---
title: "TypeScript: Prefer Interfaces"
description: "Wherever possible, use interface declarations instead of type aliases"
date: "2020-10-26T14:46:00+1000"
categories: []
keywords: []
ckTags: ["1464980"]
cardImage: "./title-card.jpeg"
---

![Stripes](title.jpeg "Photo by Markus Spiske on Unsplash")

Last week, I noticed [a Twitter thread](https://twitter.com/robpalmer2/status/1319188885197422594) from [Rob Palmer](https://twitter.com/robpalmer2) in which he described some performance problems that were caused by the use of [type alias declarations](https://www.typescriptlang.org/docs/handbook/advanced-types.html#interfaces-vs-type-aliases) in TypeScript.

Specifically, the use of a type alias declaration effected a much larger `.d.ts` output:

> Yesterday I shrank a TypeScript declaration from 700KB to 7KB by changing one line of code 🔥

In the thread, Rob points out that the reason this happens is because type alias declarations can be inlined, whereas interfaces are always referenced by name.

Let's have a look at some code that demonstrates this inlining behaviour.

Here's a [TypeScript playground snippet](https://www.typescriptlang.org/play?#code/C4TwDgpgBAShCGATAwvANmgRvAxgaygF4oAKHAewDtgJqAuKAZ2ACcBLSgcwEoiA+Jqw6cA3ACgAZgFdKOYGypQWCRCTDxgACwbN2XADRQc6LLjwM4SVBmz5eAbwC+QA) in which a type alias is used to declare the callback signature:

```ts
type ReadCallback = (content: string) => string;
function read(path: string, callback: ReadCallback) {}
```

The effected `.d.ts` output — which is shown in the playground's right-hand-side panel — looks like this:

```ts
declare type ReadCallback = (content: string) => string;
declare function read(path: string, callback: ReadCallback): void;
```

The `ReadCallback` type alias is included in the `.d.ts` output and the `read` function's signature refers to it by name.

However, the output will be different if we scope the type alias declaration — by shoving it into an [IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE) — [like this](https://www.typescriptlang.org/play?#code/MYewdgzgLgBATgUwIYBMYF4YAosEoMB8MA3gFAwxQCeADgjAErIoDCSANuwEZLADWGbKDBQEIgFwxocAJZgA5vnRFpc+QG5y8BFACucMDABmusMCgzwWGkigALSaoUAaGMA7defSU1RtOPPz4xAC+pCG4eOpAA):

```ts
const read = (() => {
  type ReadCallback = (content: string) => string;
  return function (path: string, callback: ReadCallback) {};
})();
```

The scoped type alias declaration won't be included in the output and callback's signature will be inlined:

```ts
declare const read: (
  path: string,
  callback: (content: string) => string
) => void;
```

_IIFEs aren't the only scenario in which inlining can occur — for example, type aliases that aren't exported will be inlined outside of the module in which they are declared — but they're used here so that inlining can be demonstrated in the TypeScript playground._

Inlining something that won't happen with interfaces, as interfaces are always referred to by name. If an interface is scoped in a similar manner, [like this](https://www.typescriptlang.org/play?#code/MYewdgzgLgBATgUwIYBMYF4YAosEoMB8MA3gFAwwCWYUCcAZksAjAErIoDCSANjwEZMA1iWygaCGgC4Y0ONQDmuGXMUBuGAF815eAigBXOGBj0DYYFErhsAByRQAFiqjywCgDQxgvAcJnsqNx8gsBC+MTapJq4eGpAA):

```ts
const read = (() => {
  interface ReadCallback {
    (content: string): string;
  }
  return function (path: string, callback: ReadCallback) {};
})();
```

compilation will fail with an error:

```text
Exported variable 'read' has or is using private name 'ReadCallback'.
```

Interfaces aren't inlined. They are referred to by name and if the name is private, compilation will fail.

In early versions of TypeScript, the behavioural differences between type aliases and interfaces was significant. However, in recent versions there is less of a distinction and that has led to some developers preferring type aliases over interfaces.

The performance problems mentioned in Rob's thread suggest that perhaps interfaces should be preferred. And [Daniel Rosenwasser's endorsement](https://twitter.com/drosenwasser/status/1319205169918144513) of interfaces is emphatic:

> Honestly, my take is that it should really just be interfaces for anything that they can model. There is no benefit to type aliases when there are so many issues around display/perf.
>
> We tried for a long time to paper over the distinction because of people's personal choices, but ultimately unless we actually simplify the types internally (could happen) they're not really the same, and interfaces behave better.

For developers interested ensuring that interfaces are used wherever possible, I've added a [`prefer-interface` ESLint rule](https://github.com/cartant/eslint-plugin-etc/blob/cab3ae77a8ffe41922842ea09d01733fb9fccf71/docs/rules/prefer-interface.md) to [`eslint-plugin-etc`](https://github.com/cartant/eslint-plugin-etc). It will effect a lint failure whenever it finds a type alias declaration that could be declared as an interface. The rule has a fixer — and a suggestion — and can replace type alias declarations automatically.
