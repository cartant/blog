---
title: "TIL: Overriding a Frozen Object"
description: "Frozen objects highlight some interesting property-access details"
date: "2020-10-16T13:39:00+1000"
categories: []
keywords: []
ckTags: []
cardImage: "./title-card.jpeg"
---

![Iceberg](title.jpeg "Photo by Annie Spratt on Unsplash")

When solving a problem that involved a frozen object, I learnt a bit more about the details of JavaScript object property access and some of those details are kinda interesting.

## The Problem

Recently, I replaced an ESLint rule that was specific to arrays — `no-array-foreach` — with a more general rule — `no-foreach` — that also effected failures for the `Set`, `Map` and `NodeList` types.

The `no-foreach` rule supports a `types` option — so the that the types for which it is enforced can be configured by the developer — and I planned to leverage that by deprecating and re-implementing `no-array-foreach` to use `no-foreach` internally.

Something like this:

```js{8-11}
const baseRule = require("./no-foreach");
module.exports = {
  meta: {
    /* ... */
    deprecated: true,
    replacedBy: ["no-foreach"]
  },
  create(context) {
    context.options = [{ types: ["Array"] }];
    return baseRule.create(context);
  }
};
```

Here, `context`'s `options` property is overwritten before it's passed to the base rule's `create` function. However, that fails with an error:

```text
TypeError: Error while loading rule 'no-array-foreach': Cannot assign to read only
property 'options' of object '#<Object>'
```

The property is read-only because ESLint [calls `Object.freeze`](https://github.com/eslint/eslint/blob/82669fa66670a00988db5b1d10fe8f3bf30be84e/lib/linter/linter.js#L893-L929) on `context` before passing it the the rule's `create` function.

To work around this, it's necessary to create a new object that includes the `options`, along with all of the other properties on `context`. And there are a number of ways that can be done.

## Spread Syntax

When the spread syntax is used, all of `context`'s (own) properties are spread into the new object — `contextForBaseRule` — along with the specified `options`, like this:

```js{6-12}
const baseRule = require("./no-foreach");
module.exports = {
  meta: {
    /* ... */
  },
  create(context) {
    const contextForBaseRule = {
      ...context,
      options: [{ types: ["Array"] }]
    };
    return baseRule.create(contextForBaseRule);
  }
};
```

There is a problem with this, though: it breaks the prototype chain. That's not an issue with the `options` and `report` properties — they're _own_ properties on `context` — however, `context` has a bunch of other properties that are on its prototype and, with that implementation, the rule fails with an error:

```text
Error while loading rule 'no-array-foreach': You have used a rule which requires
parserServices to be generated. You must therefore provide a value for the
"parserOptions.project" property for @typescript-eslint/parser.
```

`parserOptions` is one of the properties that's on `context`'s prototype and it's needed within the rule to retrieve the TypeScript node that corresponds to an ESLint node.

## Proxy

It's possible to use a `Proxy` to override an object's property, like this:

```js{6-16}
const baseRule = require("./no-foreach");
module.exports = {
  meta: {
    /* ... */
  },
  create(context) {
    const contextForBaseRule = new Proxy(context, {
      get(target, property, receiver) {
        if (property === "options") {
          return [{ types: ["Array"] }];
        }
        return Reflect.get(target, property, receiver);
      }
    });
    return baseRule.create(contextForBaseRule);
  }
};
```

Here, a `get` handler is specified and when a property is accessed, it checks the property name. If it's `"options"`, it returns the options that need to be passed to the base rule. Otherwise, it returns the value of `context`'s property.

However, that doesn't work either. It fails with this error:

```text
TypeError: Error while loading rule 'no-array-foreach': 'get' on proxy: property
'options' is a read-only and non-configurable data property on the proxy target but
the proxy did not return its actual value (expected '[object Array]' but got
'[object Array]')
```

The reason for this is that the [invariants](https://www.ecma-international.org/ecma-262/11.0/index.html#sec-invariants-of-the-essential-internal-methods) outlined in the ECMAScript specification are enforced by the `Proxy` implementation:

> Proxy objects maintain these invariants by means of runtime checks on the result of [handlers] invoked on the [[ProxyHandler]] object.

The [`get` handler's invariants](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/get#Invariants) are:

> The value reported for a property must be the same as the value of the corresponding target object property if the target object property is a non-writable, non-configurable own data property.
>
> The value reported for a property must be `undefined` if the corresponding target object property is a non-configurable own accessor property that has undefined as its `[[Get]]` attribute.

This is something that I'd run into before, but had forgotten. It's a small comfort, though, to know that there are limits on the havoc that can be wreaked with proxies.

## Object.create

[`Object.create`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create) can be used to create a new object which has `context` as its prototype, to which the `options` can then be assigned, like this:

```js{6-10}
const baseRule = require("./no-foreach");
module.exports = {
  meta: {
    /* ... */
  },
  create(context) {
    const contextForBaseRule = Object.create(context);
    contextForBaseRule.options = [{ types: ["Array"] }];
    return baseRule.create(contextForBaseRule);
  }
};
```

However, that won't work and will fail with this error:

```text
TypeError: Error while loading rule 'no-array-foreach': Cannot assign to read only
property 'options' of object '#<Object>'
```

This surprised me. I expected the `options` property to be added to the created object — regardless of the fact that there is an `options` property on the prototype.

It turns out that that's not how the language works.

The details are in the [ECMAScript specification](https://www.ecma-international.org/ecma-262/11.0/index.html#sec-ordinary-object-internal-methods-and-internal-slots-set-p-v-receiver), but the gist of it is that when an object property is assigned a value, the runtime looks for a property descriptor to determine whether or not the assignment is permitted. First, it looks at the object's own property descriptors, but if it doesn't find a descriptor, it then looks at the object's prototype's descriptors.

So what's happening here is:

- The runtime looks for a property descriptor for the `options` property on `contextWithOption`, but it doesn't find one.
- It then looks for a property descriptor on `context` and finds one.
- However, the property descriptor indicates that the property is not writable, so an error is thrown.

Fortunately, the second parameter to `Object.create` can be used to add properties to the created object — using property descriptors like those passed to [`Object.defineProperty`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) — like this:

```js{6-14}
const baseRule = require("./no-foreach");
module.exports = {
  meta: {
    /* ... */
  },
  create(context) {
    const contextForBaseRule = Object.create(context, {
      options: {
        value: [{ types: ["Array"] }],
        writable: false
      }
    });
    return baseRule.create(contextForBaseRule);
  }
};
```

Once that's done the rule works:

- When the base rule accesses the `options` property, it finds the property on `contextForBaseRule` — which overrides the `options` property on `context`.
- When the base rule calls the `report` method, it finds the method on `context` — which is the prototype for `contextForBaseRule`.
- And when the base rule accesses the `parserOptions` property, it finds the property on `context` object's prototype.

For something that I'd expected to be straightforward, a surprising number of attempts were needed to get this working. 😅 However, I did learn some things along the way.
