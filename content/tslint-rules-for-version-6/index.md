---
title: "RxJS: TSLint Rules for Version 6"
description: "TSLint rules for clean, RxJS-version-6 code"
date: "2018-04-27T08:32:17.709Z"
categories: []
keywords: []
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-tslint-rules-for-version-6-d10e2482292d
---

![Surveillance cameras](title.jpeg "Photo by Scott Webb on Unsplash")

Earlier this week, RxJS version 6 was released and, with its release, managing RxJS imports has become much, much easier.

Last year, I wrote a bunch of TSLint rules for [managing RxJS imports](/managing-rxjs-imports-with-tslint/). They’re distributed in the [`rxjs-tslint-rules`](https://github.com/cartant/rxjs-tslint-rules) package.

Most of the package’s import-related rules are no longer required when linting an RxJS-version-6 codebase, so if the latest release of the rules finds RxJS version 6 installed in `node_modules`, the rules that are no longer required are deprecated.

If RxJS version 5 is installed — or if [`rxjs-compat`](https://github.com/ReactiveX/rxjs/blob/7cff11cc20bfb2aa5c496576501d5889da1dcb4d/MIGRATION.md) is installed alongside RxJS version 6 — the rules will behave as they did in prior releases of the package.

Let’s look briefly at what’s changed and then look at some rules that are still useful for maintaining a clean, RxJS-version-6 codebase.

## What’s changed with the imports?

The [migration guide](https://github.com/ReactiveX/rxjs/blob/7cff11cc20bfb2aa5c496576501d5889da1dcb4d/MIGRATION.md) discusses all of the changes in detail, but the main, import-related changes can be summarised as follows:

- Classes, types and observable factory functions are imported from `"rxjs"`.
- Operators are imported from `"rxjs/operators"`.
- Factory functions and operators that patch `Observable` and `Observable.prototype` have been removed.

That means that imports like these:

```ts
import { Observable } from "rxjs/Observable";
import { concat } from "rxjs/observable/concat";
import { map } from "rxjs/operators/map";
import { take } from "rxjs/operators/take";
```

Are replaced with imports like these:

```ts
import { concat, Observable } from "rxjs";
import { map, take } from "rxjs/operators";
```

And it means that patching imports like these are no longer supported:

```ts
import "rxjs/add/observable/concat";
import "rxjs/add/operator/map";
import "rxjs/add/operator/take";
```

This greatly simplifies dealing with RxJS imports and renders many of the import-related rules in `rxjs-tslint-rules` redundant. Woohoo!

Let’s have look at some rules that are still useful for linting an RxJS-version-6 codebase.

## Don’t touch the internals

Even though RxJS version 6 has only just been released and even though everyone knows you shouldn’t touch a package’s internals, I’ve already seen codebases that have imports like this:

```ts
import { map } from "rxjs/internals/operators/map";
```

It’s safe to say that imports that rely upon the package’s internal structure will break, eventually. So if you work with developers that live the [YOLO](https://en.m.wikipedia.org/wiki/YOLO_%28aphorism%29) life, you might find the `rxjs-no-internal` rule useful.

```json
"rxjs-no-internal": {
  "severity": "error"
}
```

## Preparing for further change

RxJS version 6 deprecates a number of library features that will be removed or changed in version 7. In particular:

- the `combineLatest`, `concat`, `merge`, `race` and `zip` operators are deprecated in favour of their namesake factory functions;
- the result selector parameter accepted by numerous operators is deprecated in favour of using an additional `map` operator; and
- some parts of the library have been deprecated for internal use only.

TSLint has a built-in rule that can be used to effect a warning — or an error — if deprecated functionality is used:

```json
"deprecation": {
  "severity": "warning"
}
```

With a large codebase, the number of deprecations can be overwhelming.

If you want to update your codebase in a piecemeal fashion, you can use the `rxjs-ban-observables` and `rxjs-ban-operators` rules, adding only what you want to ban to the rules’ options.

```json
"rxjs-ban-observables": {
  "options": [{
    "empty": "Use EMPTY",
    "never": "Use NEVER",
  }],
  "severity": "error"
},
"rxjs-ban-operators": {
  "options": [{
    "combineLatest": "Use the static combineLatest",
    "concat": "Use the static concat",
    "merge": "Use the static merge",
    "race": "Use the static race",
    "zip": "Use the static zip",
  }],
  "severity": "error"
}
```

## Do you speak Finnish?

Some developers love [Finnish notation](https://medium.com/@benlesh/observables-and-finnish-notation-df8356ed1c9b) and some developers loathe it. If you love it, you can enforce its use using the `rxjs-finnish` rule:

```json
"rxjs-finnish": {
  "severity": "error"
}
```

By default, the rule will enforce the use of Finnish notation for functions, methods, parameters, properties and variables. If you only want to enforce Finnish notation for, say, variables you can do so using the rule’s options, like this:

```json
"rxjs-finnish": {
  "options": [{
    "functions": false,
    "methods": false,
    "parameters": false,
    "properties": false,
    "variables": true
  }],
  "severity": "error"
}
```

If you loathe Finnish notation, you can disallow its use using the `rxjs-no-finnish` rule:

```json
"rxjs-no-finnish": {
  "severity": "error"
}
```

## Subjects

The package contains a couple of rules that deal with subjects.

The `rxjs-no-subject-unsubscribe` can be used to disallow the calling of `unsubscribe` on subjects, as calling it can effect some [surprising behaviour](/closed-subjects/).(Also, in version 7, the `unsubscribe` method is likely to be removed from the `Subject` class.)

```json
"rxjs-no-subject-unsubscribe": {
  "severity": "error"
}
```

The `rxjs-no-subject-value` rule can be used to disallow the use of a `BehaviorSubject`’s `value` property — which some developers consider to be a code smell.

```json
"rxjs-no-subject-value": {
  "severity": "error"
}
```

## Careful now

The package includes a couple of rules that can help avoid potentially-unsafe practices.

The `rxjs-no-unsafe-scope` rule can be used to disallow the accessing or manipulation of state that’s external to a callback function.

```json
"rxjs-no-unsafe-scope": {
  "severity": "error"
}
```

And the `rxjs-no-unsafe-switchmap` rule can be used to disallow the use of `switchMap` in NgRx effects — and `redux-observable` epics — into which it is likely — based on the action type — to [introduce race conditions](/avoiding-switchmap-related-bugs/).

```json
"rxjs-no-unsafe-switchmap": {
  "severity": "error"
}
```

## Errors

If you’re like me and only want `Error`\-derived objects to be passed to an observer’s error method, there’s a rule to enforce that:

```json
"rxjs-throw-error": {
  "severity": "error"
}
```

## Upgrading from 5 to 6

Another package of RxJS-related TSLint rules — written by some of the developers at Google — was released in conjunction with RxJS version 6: [`rxjs-tslint`](https://github.com/ReactiveX/rxjs-tslint).

Its rules take advantage of TSLint’s automated fixing and can be used to automatically:

- update RxJS imports;
- replace dot-chained operators with pipeable operators; and
- replace static `Observable` method calls with factory function calls.

If you’re updating a codebase from 5 to 6, you should check it out. It’s awesome.
