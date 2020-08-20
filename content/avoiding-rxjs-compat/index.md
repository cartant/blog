---
title: "RxJS: Avoiding rxjs-compat"
description: "How to avoid unintentional rxjs-compat dependencies"
date: "2019-02-17T07:02:55.500Z"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title.jpeg"
slug: "/@cartant/rxjs-avoiding-rxjs-compat-4b79a566359b"
---

![Stacked cups](title.jpeg "Photo by Monika Stawowy on Unsplash")

Introducing unintentional dependencies on `rxjs-compat` is something that I see developers doing every now and then. Let’s have a look at `rxjs-compat` to see what it is, how it works and how depending upon it can be avoided.

_If you’re only interested in avoiding the dependency, skip to the TL;DR at the bottom of the article._

## So what is it?

In RxJS version 6, breaking changes made the library simpler:

- the prototype-patching operators were removed; and
- the export locations were rearranged so that each export was available from only a single location.

Those changes made the library easier to maintain, document and explain, but they created a burden for developers with large RxJS-version-5 codebases: pretty much all RxJS-related code would need to be modified.

The `rxjs-compat` package was created so that developers could upgrade to RxJS version 6 without having to modify code. It re-implements the prototype-patching operators and makes available all of the RxJS-version-5 export locations.

## How does it work?

The RxJS-version-6 distribution includes files for all of the version-5 export locations. However, the imports within those files redirect to `rxjs-compat`.

For example, let’s look at the `of` observable creator and the `mapTo` operator.

In RxJS version 5, you can choose to import everything:

```ts
import Rx from "rxjs/Rx";
const answer = Rx.Observable.of(6 * 9).mapTo(42);
```

Or you can choose to patch `Observable` with only the creators and operators you need:

```ts
import { Observable } from "rxjs/Observable";
import "rxjs/add/observable/of";
import "rxjs/add/operator/mapTo";
const answer = Observable.of(6 * 9).mapTo(42);
```

Or you can choose to directly import only the creators and operators you need:

```ts
import { of } from "rxjs/observable/of";
import { mapTo } from "rxjs/operator/mapTo";
const answer = mapTo.call(of(6 * 9), 42);
```

With `rxjs-compat` installed, all of the above snippets will work with RxJS version 6 — despite their using version-5 export locations.

So how does this work? Well, the RxJS-version-6 package includes all of the files imported in the snippets:

```text
node_modules/rxjs/Rx.js
node_modules/rxjs/Observable.js
node_modules/rxjs/add/observable/of.js
node_modules/rxjs/add/operator/mapTo.js
node_modules/rxjs/observable/of.js
node_modules/rxjs/operator/mapTo.js
```

But each of those files contains nothing apart from the import of an `rxjs-compat` file. For example, `rxjs/Observable.js` looks like this:

```js
"use strict";
function __export(m) {
  for (var p in m) if (!exports.hasOwnProperty(p)) exports[p] = m[p];
}
Object.defineProperty(exports, "__esModule", { value: true });
__export(require("rxjs-compat/observable/of"));
//# sourceMappingURL=of.js.map
```

And `rxjs-compat/observable/of.js` looks like this:

```js
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
var rxjs_1 = require("rxjs");
exports.of = rxjs_1.of;
//# sourceMappingURL=of.js.map
```

So importing `of` from a version-5 export location, imports `of` from `rxjs-compat` which imports everything from `rxjs`.

## How can it be avoided? — TL;DR

If an RxJS-version-5 export location us used — without having `rxjs-compat` installed — an error something like this is effected:

```text
ERROR in ~/project/node_modules/rxjs/observable/of.js
Module not found: Error: Can't resolve 'rxjs-compat/observable/of'
in '~/project/node_modules/rxjs/observable'
```

Which is a little confusing — unless you know how `rxjs-compat` works — so it’s understandable that developers sometimes resort to installing `rxjs-compat` to resolve the error. It’s likely to continue to cause confusion, too, as there are so many snippets — on Stack Overflow and elsewhere — that use version-5 export locations.

The preferred way to resolve the error is to import only from RxJS-version-6 export locations. Fortunately, there are only a few:

```ts
import /* classes, creators, schedulers */ "rxjs";
import { ajax } from "rxjs/ajax";
import /* pipeable operators */ "rxjs/operators";
import { TestScheduler } from "rxjs/testing";
import { webSocket } from "rxjs/webSocket";
```

So, in RxJS version 6, the snippets above should be rewritten as:

```ts
import { of } from "rxjs";
import { mapTo } from "rxjs/operators";
const answer = of(6 * 9).pipe(mapTo(42));
```

To help avoid the problem, I’ve added a TSLint rule — `rxjs-no-compat`— to [`rxjs-tslint-rules`](https://github.com/cartant/rxjs-tslint-rules) that bans imports from RxJS-version-5 export locations.
