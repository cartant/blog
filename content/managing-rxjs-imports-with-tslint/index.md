---
title: Managing RxJS Imports with TSLint
description: How to use TSLint to enforce RxJS import policies
date: "2017-07-21T01:14:07.242Z"
categories: []
keywords: []
cardImage: "./title.jpeg"
slug: /@cartant/managing-rxjs-imports-with-tslint-828cdc66b5ee
---

![Jigsaw puzzle](title.jpeg)

There are a number of options for importing RxJS and these are detailed in the [documentation](http://reactivex.io/rxjs/manual/installation.html).

You can import everything:

```ts
import Rx from "rxjs/Rx";

Rx.Observable.of(1, 2, 3).map(i => i.toString());
```

You can import only the methods you need, patching the `Observable` prototype:

```ts
import { Observable } from "rxjs/Observable";
import "rxjs/add/observable/of";
import "rxjs/add/operator/map";

Observable.of(1, 2, 3).map(i => i.toString());
```

Or you can import the methods to be called directly — not via the `Observable` prototype:

```ts
import { Observable } from "rxjs/Observable";
import { of } from "rxjs/observable/of";
import { map } from "rxjs/operator/map";

const source = of(1, 2, 3);
const mapped = map.call(source, i => i.toString());
```

Each of these options has its advantages and disadvantages:

- importing everything is simple and ensures all methods are available, but comes at the cost of importing the entire 586 KB (unminified) bundle;
- patching `Observable` with only the methods you wish to use results in a smaller build, but care must be taken to ensure that methods are imported before they are used and that all necessary methods are imported;
- importing and calling methods directly avoids the possibility of calling a method that has not been patched into the `Observable` prototype, but comes at the cost of more verbose code and lost type information (as, at this point in time, `Function.prototype.call` returns `any`).

It makes sense to adopt one of the above options as a general policy for a code base. The enforcement of such a policy can be managed using TSLint and some [custom rules](https://palantir.github.io/tslint/develop/custom-rules/).

## Rules

I’ve compiled a small set of custom, import-related rules; they are included in the [`rxjs-tslint-rules`](https://github.com/cartant/rxjs-tslint-rules) package.

The rules are fine-grained and can be combined in various ways to enforce numerous RxJS import polices.

## Rules for the Angular Policy

The Angular approach is to import methods directly and to use `call` to invoke them on observable instances. One of the motivations for this approach is to avoid breaking client code between Angular updates. If Angular were to patch the `Observable` prototype and if clients were to rely upon methods being present solely due to Angular’s patching of the prototype, client code would break if Angular ceased to patch a particular method.

The rule combination looks like this:

```json
"rxjs-no-add": { "severity": "error" },
"rxjs-no-patched": { "severity": "error" },
"rxjs-no-wholesale": { "severity": "error" }
```

`rxjs-no-add` disallows imports that patch `Observable.prototype`; `rxjs-no-patched` disallows calls to methods via the prototype; and `rxjs-no-wholesale` disallows the importation of the entire RxJS library.

A custom rule to disallow unused, directly-imported methods is not required; just use TSLint’s [`no-unused-variable`](https://palantir.github.io/tslint/rules/no-unused-variable/) rule.

## Rules for the Patch-in-every-file Policy

In shared code bases in which files might be imported individually, a sensible policy is to enforce the importation of patched methods in every file in which they are used.

The rule combination looks like this:

```json
"rxjs-add": { "severity": "error" },
"rxjs-no-unused-add": { "severity": "error" },
"rxjs-no-wholesale": { "severity": "error" }
```

`rxjs-add` ensures that any methods used in a module are imported in that module; `rxjs-no-unused-add` disallows the importation of methods that are not used; and `rxjs-no-wholesale` disallows the importation of the entire RxJS library.

## Rules for the Patch-in-one-file Policy

Another common policy is to nominate a particular file and import patched methods into that file only.

The rule combination looks like this:

```json
"rxjs-add": {
  "options": \[{
    "file": "./source/patched-imports.ts"
  }\],
  "severity": "error"
},
"rxjs-no-wholesale": { "severity": "error" }
```

`rxjs-add` ensures that methods that patch `Observable.prototype` are imported only in the `./source/patched-imports.ts` file and disallows the importation of methods that are not used; and `rxjs-no-wholesale` disallows the importation of the entire RxJS library.

## Configuration

To use the custom rules, add `rxjs-tslint-rules` to the `extends` setting in your `tslint.json` file and add the rules use want to use to the `rules` setting:

```json
{
  "extends": \[
    "rxjs-tslint-rules"
  \],
  "rules": {
    "rxjs-add": { "severity": "error" },
    "rxjs-no-unused-add": { "severity": "error" },
    "rxjs-no-wholesale": { "severity": "error" }
  }
}
```

`rxjs-tslint-rules` is not opinionated and contains no enabled, default rules. The rules are explained in more detail in the [documentation](https://github.com/cartant/rxjs-tslint-rules/blob/master/README.md).

Most of the rules require [type checking](https://palantir.github.io/tslint/usage/type-checking/) — which means that TSLint will need to compile the program. More detailed information about configuring TSLint can be found [here](https://palantir.github.io/tslint/usage/configuration/).

---

With the release of RxJS version 5.5.0, there is another alternative: importing and using pipeable (also known as lettable) operators.

Pipeable operators have several advantages and if you wish to switch to them, `rxjs-tslint-rules` can be used enforce a pipeable-operators-only policy. The rule combination for the policy is detailed in my article [_RxJS: Understanding Lettable Operators_](/understanding-lettable-operators/).
