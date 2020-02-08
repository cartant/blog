---
title: "TIL: String.raw"
description: How to avoid escaping the escape character in JavaScript string literals
date: "2019-12-21T11:22:00+1000"
categories: []
keywords: []
cardImage: "./title.jpeg"
---

![Ingredients](title.jpeg "Photo by Icons8 Team on Unsplash")

Something that's always annoyed me — because I always seem to mess it up — is having to escape the escape character (`\`) in some JavaScript string literals.

Well, it turns out there's a feature that means having to do this can be avoided entirely.

## Other languages

C# uses the `@` character as a verbatim identifier and when it's applied to a string literal, it works like this:

```csharp
// These strings are equivalent:
const verbatim = @"C:\Windows\system.ini";
const escaped = "C:\\Windows\\system.ini";
```

This is part of the C# language syntax.

## JavaScript

In JavaScript, the mechanism is a little different. It doesn't extend the syntax. Instead, it leverages template literals and their tags.

The `String` object has a static [`raw`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/raw) method that is intended to be used as a [template-literal tag](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#Tagged_templates), like this:

```js
// These strings are equivalent:
const raw = String.raw`C:\Windows\system.ini`;
const escaped = "C:\\Windows\\system.ini";
```

Having to put backslash-delimited Windows paths in JavaScript literals isn't something that I've ever had to do.

However, I often have to escape escape characters when I'm dynamically creating a regular expression using the `RegExp` constructor. And in these situations, not having to escape the escape characters makes things much clearer:

```js
const regExp = new RegExp(String.raw`(\b|_)${word}(\b|_)`));
```

## ES2018

It's worth noting that there are differences in how these 'raw' template literals are handled in [different JavaScript runtimes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#Tagged_templates_and_escape_sequences).

In ES2016, this template literal will effect an error, because `\u` is interpreted as a Unicode escape sequence:

```js
const directory = String.raw`C:\images\unsplash`;
```

In ES2018, this behaviour was changed and — as long as a tag function is applied — no error will be effected.

## More template-literal tags

If template-literal tags are something you've not encountered before, you might also want to check out the [`common-tags`](https://github.com/declandewet/common-tags) package. It has a whole bunch of extremely useful tag functions.
