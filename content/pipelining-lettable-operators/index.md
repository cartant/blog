---
title: "RxJS: Pipelining Lettable Operators"
description: A look at the TC39 pipeline operator proposal
date: "2017-09-29T11:18:52.660Z"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-pipelining-lettable-operators-f92f6843d817
---

![Effects pedals](title.jpeg "Photo by David Rangel on Unsplash")

Earlier this week, a TC39 proposal for a [pipeline operator](https://github.com/tc39/proposal-pipeline-operator) moved to [stage-1](https://tc39.github.io/process-document/). If the proposal is eventually accepted and included in the ECMAScript standard — it has a long way to go — it will offer a new syntax for [lettable operators](/understanding-lettable-operators/).

## What is proposed?

The proposed pipeline operator is `|>`. It’s a binary operator; the operand to its left is a value and the operand to its right is a function. The pipeline operator calls the function, passing the value as an argument, and returns the function’s result.

That is, `64 |> Math.sqrt` is equivalent to `Math.sqrt(64)`.

Using the pipeline operator, multiple functions can be pipelined, like this:

```ts
function join(array) {
  return array.join(", ");
}

function capitalize(text) {
  return `${text[0].toUpperCase()}${text.substring(1)}`;
}

function exclaim(text) {
  return `${text}!`;
}

const result = ["hello", "world"]
  |> join
  |> capitalize
  |> exclaim;

console.log(result); // "Hello, world!"
```

## How would it be used with lettable operators?

The lettable operators introduced in the RxJS 5.5.0 beta are higher-order functions. They return functions that receive and return observables. As such, they can be used with the proposed pipeline operator, like this:

```ts
import { range } from "rxjs/observable/range";
import { map, filter, scan, toArray } from "rxjs/operators";

const value$ = range(0, 10)
  |> filter(x => x % 2 === 0)
  |> map(x => x + x)
  |> scan((acc, x) => acc + x, 0)
  |> toArray();

value$.subscribe(x => console.log(x)); // [0, 4, 12, 24, 40]
```

## When?

Whether or not the pipeline proposal will navigate its way through the TC39 process to be included in the ECMAScript standard remains to be seen.

However, if you like to live at the bleeding edge, there is a Babel plugin for the proposal: [`babel-plugin-transform-pipeline`](https://github.com/SuperPaintman/babel-plugin-transform-pipeline). It does have [some issues](https://github.com/SuperPaintman/babel-plugin-transform-pipeline/issues/1) with recent Babel packages, but with the proposal at stage-1, it should not be long before an up-to-date plugin is included in Babel’s [stage-1 preset](https://github.com/babel/babel/tree/master/packages/babel-preset-stage-1).

Regarding TypeScript, the pipeline proposal won’t be considered [until it reaches stage-2](https://twitter.com/SeaRyanC/status/913134791230287872) (or maybe [stage-3](https://twitter.com/drosenwasser/status/959589077686218755)).
