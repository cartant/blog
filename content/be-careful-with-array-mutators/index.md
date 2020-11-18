---
title: "Be Careful with Array Mutators"
description: "Assigning mutated arrays can lead to surprises"
date: "2019-02-21T22:04:44.988Z"
categories: []
keywords: []
ckTags: []
cardImage: "./title.jpeg"
slug: "/@cartant/be-careful-with-array-mutators-2d5231351cf8"
---

![Different pumpkins](title.jpeg "Photo by Corinne Kutz on Unsplash")

A while ago, I wrote a linting rule to highlight an error that I’d made on a few occasions. Let’s look at the error — and its variations — and then at the rule that prevents it.

## The error

If you are used to working with [fluent APIs](https://www.martinfowler.com/bliki/FluentInterface.html) — perhaps with [D3](https://github.com/d3/d3) or [lodash](https://github.com/lodash/lodash) — it feels natural to write `Array`\-based code like this:

```ts
const developers = employees
  .filter((e) => e.role === "developer")
  .map((e) => e.name)
  .sort();
```

Which is fine. There’s no error in that code.

However, in situations in which you don’t need to filter or map, you might write something like this:

```ts
const sorted = names.sort();
```

It feels natural, but there’s a potential error in there. And it’s an error that can effect hard-to-find bugs. It’s also an error that’s easy to overlook in code reviews.

The `sort` method mutates the array. It sorts the array in place and returns a reference to the array itself. It does not return a sorted copy of the array. So the `sorted` and `names` variables both reference the same, sorted array. And it means that — unless the array was already sorted — any code using the `names` variable will be dealing with an array in which the elements that been rearranged.

So why is the code in the first snippet okay? It’s because `sort` is applied to the array returned by `map` — a transient array to which there are no references.

## The rule

The fundamental problem is the use of the return value when the `sort` method is applied to an array that is referenced elsewhere in the program.

If the return value is not used, it’s fine — as the intent is clear:

```ts
names.sort();
```

And it’s also fine if the `sort` method is applied to an array that has no referencing variable:

```ts
const names = ["bob", "alice"].sort();
```

It is a potential problem if the result is assigned to another variable:

```ts
const sorted = names.sort();
```

Or assigned to a property:

```ts
const data = {
  names: names.sort()
};
```

Or passed as an argument:

```ts
console.log(names.sort());
```

The rule I wrote is named `no-assign-mutated-array` and is in the `tslint-etc` package. The source code is [here](https://github.com/cartant/tslint-etc/blob/master/source/rules/noAssignMutatedArrayRule.ts).

It works by searching for method calls — like `sort` — that mutate arrays and checking to see if they are statements. The search is done using [Craig Spence](https://twitter.com/phenomnominal)’s awesome [`tsquery`](https://github.com/phenomnomnominal/tsquery).

If the call is a statement, the return value isn’t used — so there is no potential problem.

If the call is not a statement, the rule determines whether or not the array upon which the call is made is transient and effects a rule failure if it isn’t.

There are several methods on `Array.prototype` that mutate the array and return a reference to the mutated array — `fill`, `reverse` and `sort` — and the rule is enforced for all of them.

## The future

In the [TypeScript roadmap ](https://github.com/Microsoft/TypeScript/issues/29288)— for January to June 2019 — it was announced that TypeScript will be switching to ESLint — using [`@typescript-eslint/parser`](https://github.com/typescript-eslint/typescript-eslint).

So sometime soon, I’ll start converting the rules in `tslint-etc` and `rxjs-tslint-rules` from TSLint rules to ESLint rules. Doing that should be … interesting. And I imagine it’ll prompt me to write a few more linting articles.
