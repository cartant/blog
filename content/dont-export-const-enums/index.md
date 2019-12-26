---
title: "TypeScript: Don’t Export const enums"
description: Exporting const enums from libraries will break some users
date: "2019-12-14T10:36:00+1000"
categories: []
keywords: []
---

![Brick wall](title.jpeg "Photo by Waldemar Brandt on Unsplash")

If you are writing a library and you export a `const enum`, some developers will not be able to compile their applications if they import your library. Let’s look at why.

## Non-const enums

When you declare an `enum`, TypeScript will generate code for it. For example, this TypeScript snippet:

```ts
enum Bool {
  True,
  False,
  FileNotFound,
}
let value = Bool.FileNotFound;
```

will compile to this JavaScript:

```ts
var Bool;
(function(Bool) {
  Bool[(Bool["True"] = 0)] = "True";
  Bool[(Bool["False"] = 1)] = "False";
  Bool[(Bool["FileNotFound"] = 2)] = "FileNotFound";
})(Bool || (Bool = {}));
let value = Bool.FileNotFound;
```

The reasons for this are explained in the [documentation](www.typescriptlang.org/docs/handbook/enums.html). However, some developers don’t need the features provided by this style of declaration and don’t want the costs involved — they just want to use enums instead of constants.

## const enums

When an `enum` is declared as `const`, TypeScript doesn’t generate code for the declaration. For example, this TypeScript snippet:

```ts
const enum Bool {
  True,
  False,
  FileNotFound,
}
let value = Bool.FileNotFound;
```

will compile to this JavaScript:

```ts
let value = 2 /* FileNotFound */;
```

No code is generated for the `enum` declaration. Which is great — it’s just like using a constant — but there is a problem.

## Isolated modules

In the above snippets, TypeScript has access to the `const enum` declaration, as it’s in the same module as the declaration for the `value` variable.

However, if the `const enum` declaration is in a different module — and is imported into the module that contains the variable declaration — TypeScript will have to read both modules to determine that `Bool.FileNotFound` should be replaced with `2`.

This is a problem because some developers use a workflow that separates type checking from compilation — with compilation happening on an isolated-module basis:

- the compiler reads a TypeScript module;
- the module’s type information is stripped; and
- what’s left is the JavaScript module that the compiler writes.

This compilation process does not read imported modules, so it’s not possible for it to support the replacement of `const enum` members — like `Bool.FileNotFound` — with their values.

The [`transpileModule`](https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API#a-simple-transform-function) function in the TypeScript compiler API performs this type of compilation, as does [`@babel/plugin-transform-typescript`](https://babeljs.io/docs/en/babel-plugin-transform-typescript) — which is what’s used in [`create-react-app`](https://github.com/facebook/create-react-app).

TypeScript has an `isolatedModules` compiler option that performs additional checks to ensure that the compiled code is safe for this type of compilation process. If you are are writing a library, you should enable this option to ensure that you are not exporting `const enum` declarations and that all TypeScript developers can compile code that imports your library.
