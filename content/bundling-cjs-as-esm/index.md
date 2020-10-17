---
title: "Bundling CommonJS into an ES Module"
description: "Using Rollup to distribute CJS within ESM"
date: "2020-08-19T16:49:00+1000"
categories: []
keywords: []
ckTags: []
cardImage: "./title-card.jpeg"
---

![Box](title.jpeg "Photo by Kelli McClintock on Unsplash")

Recently, I've made some changes to the [`rxjs-spy`](https://github.com/cartant/rxjs-spy) distribution — so that it's consumable as an ES module — and I'm documenting the process as a blog post in case I need to do something like this again.

The problem surfaced with the release of Angular 10. When a non-ES module dependency is consumed, the Angular CLI effects the following warning:

```text
WARNING in some-app depends on 'rxjs-spy/operators'.
CommonJS or AMD dependencies can cause optimization bailouts.
For more info see:
https://angular.io/guide/build#configuring-commonjs-dependencies
```

To prevent this, the `rxjs-spy` distribution needs to include a `module` entry in its `package.json` — referencing an ES module export.

Generating ES module output from the TypeScript source is straightforward and most of my libraries are distributed with CommonJS and ES module exports. However, there was a problem: `rxjs-spy` depends upon several packages that do not provide ES module exports.

I wanted to spend as little time as possible solving the problem — as `rxjs-spy` is soon going to be replaced with [RxJS Tools](https://rxjs.tools) — so I decided to use Rollup to generate two ES module bundles: one for the spy infrastructure and one for the operators.

This bundled approach means that the CommonJS modules depended upon by `rxjs-spy` are included in the ES module bundles in their entirety. It's relatively simple, but it has a downside: if the application itself consumes the CommonJS modules bundled within `rxjs-spy` it will contain duplicated code — as the modules won't be shared.

However, it's not really an issue here, as the CommonJS modules are only used in the spy bundle — not the operator bundle — and the spy bundle essentially development-build-only.

The Rollup configuration file for the spy bundle looks like this:

```js
import babel from "@rollup/plugin-babel";
import commonjs from "@rollup/plugin-commonjs";
import json from "@rollup/plugin-json";
import replace from "@rollup/plugin-replace";
import resolve from "@rollup/plugin-node-resolve";
import pack from "./package.json";

const extensions = [".js", ".ts"];

export default {
  external: ["rxjs", "rxjs/operators"],
  input: "source/index.ts",
  output: [
    {
      file: "dist/esm/index.js",
      format: "esm",
      sourcemap: true
    }
  ],
  plugins: [
    json(),
    resolve({ extensions }),
    commonjs(),
    babel({ babelHelpers: "bundled", extensions }),
    replace({
      __RX_SPY_VERSION__: `"${pack.version}"`
    })
  ]
};
```

The configuration is relatively straightforward:

- `@rollup/plugin-json` enables the consumption of JSON files as modules.
- `@rollup/plugin-node-resolve` enables `node_modules` consumption.
- `@rollup/plugin-commonjs` enables the consumption of CommonJS modules.
- `@rollup/plugin-babel` compiles the TypeScript files to JavaScript.
- `@rollup/plugin-replace` performs a string replacement of a version placeholder.
- The `externals` setting ensures that RxJS imports are left intact, so that RxJS is modules are not included in the ES bundle.

The configuration for the operators bundle is similar. The only real difference is that it doesn't require the string replacement.

In the distribution, the spy bundle is located at `esm/index.js`, so the `package.json` contains a `module` setting:

```json
{
  "module": "esm/index.js"
}
```

The operators bundle is located at `esm/operators/index.js` and to enable imports like this:

```ts
import { tag } from "rxjs-spy/operators";
```

there is a `operators/package.json` with this content:

```json
{
  "main": "../cjs/operators/index.js",
  "module": "../esm/operators/index.js",
  "types": "../cjs/operators/index.d.js"
}
```

When a bundler sees `rxjs-spy/operators` in an import statement it will read the `package.json` and be redirected to either the CommonJS module, the ES module or the TypeScript type definitions, depending upon what it's looking for.

Lastly, in order to avoid breaking changes, the distribution needs to support imports like this:

```ts
import { tag } from "rxjs-spy/operators/tag";
```

It manages this by including sub-directories for each operator, with those sub-directories each containing a `package.json` file to redirect bundlers to the appropriate file.

For example, the `operators/tag/package.json` file has this content:

```json
{
  "main": "../../cjs/operators/tag.js",
  "module": "../esm.js",
  "types": "../../cjs/operators/tag.d.js"
}
```

And the `operators/tag/esm.js` file — to which the bundler is redirected — has this content:

```js
export { tag } from "..";
```

Which imports the `tag` operator from the ES bundle in the parent directory and then re-exports it.

This `package.json` shenanigans will get a little simpler when the `exports` property is more widely supported. If you're interested, you can read more about it [here](https://nodejs.org/api/esm.html#esm_package_entry_points).
