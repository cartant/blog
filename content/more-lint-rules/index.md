---
title: "More Lint Rules"
description: "Two more ESLint rules: one for Typescript and one for React"
date: "2020-08-19T12:58:00+1000"
categories: []
keywords: []
ckTags: ["1464980", "1464981"]
cardImage: "./title-card.jpeg"
---

![Maginfying glass](title.jpeg "Photo by Markus Winkler on Unsplash")

Last week, I created two more ESLint rules.

The first is an ESLint-port of a TypeScript-specific Wotan rule — Wotan is [yet another linter](https://github.com/fimbullinter/wotan/blob/11368a193ba90a9e79b9f6ab530be1b434b122de/README.md#why-yet-another-linter) — named [`no-misused-generics`](https://github.com/fimbullinter/wotan/blob/11368a193ba90a9e79b9f6ab530be1b434b122de/packages/mimir/docs/no-misused-generics.md). It's a rule that [Oliver Ash](https://twitter.com/OliverJAsh) mentioned in a conversation that stemmed a tweet of [Dan Vanderkam](https://twitter.com/danvdk)'s:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">The Golden Rule of Generics in <a href="https://twitter.com/typescript?ref_src=twsrc%5Etfw">@typescript</a> (probably other languages, too!) h/t to <a href="https://twitter.com/SeaRyanC?ref_src=twsrc%5Etfw">@SeaRyanC</a> and the new TS handbook for this amazingly concrete rule. <a href="https://t.co/Z3ETr1MQ22">https://t.co/Z3ETr1MQ22</a></p>&mdash; Dan Vanderkam (@danvdk) <a href="https://twitter.com/danvdk/status/1293635068930543616?ref_src=twsrc%5Etfw">August 12, 2020</a></blockquote>

You should read Dan's blog post. It explains the reasoning behind the rule.

The port was simple, as the Wotan rule's implementation has a single top-level function. All that had to be done was to call that function whenever ESLint encounters an AST node that has a call TypeScript signature:

<!-- prettier-ignore -->
```ts
return {
  ArrowFunctionExpression: checkSignature,
  FunctionDeclaration: checkSignature,
  FunctionExpression: checkSignature,
  MethodDefinition: checkSignature,
  "Program:exit": () => (usage = undefined),
  TSCallSignatureDeclaration: checkSignature,
  TSConstructorType: checkSignature,
  TSConstructSignatureDeclaration: checkSignature,
  TSDeclareFunction: checkSignature,
  TSFunctionType: checkSignature,
  TSIndexSignature: checkSignature,
  TSMethodSignature: checkSignature,
  TSPropertySignature: checkSignature,
};
```

and then obtain the corresponding TypeScript AST node via the `esTreeNodeToTSNodeMap` map that's provided by the [`@typescript-eslint/parser`](https://github.com/typescript-eslint/typescript-eslint/tree/daf649f6e7f63353a332a21b4fa79cb376de37eb/packages/parser):

<!-- prettier-ignore -->
```ts{1,5}
const { esTreeNodeToTSNodeMap } = getParserServices(context);
let usage: Map<ts.Identifier, tsutils.VariableInfo> | undefined;

function checkSignature(node: es.Node) {
  const tsNode = esTreeNodeToTSNodeMap.get(node);
  if (
    tsutils.isSignatureDeclaration(tsNode) &&
    tsNode.typeParameters !== undefined
  ) {
    checkTypeParameters(tsNode.typeParameters, tsNode);
  }
}
```

With the rule enabled, misused generics like this:

<!-- prettier-ignore -->
```ts
function parseYAML<T>(input: string): T {
  /* parsing implementation */
}
```

will effect a lint failure:

```text
Type parameter 'T' cannot be inferred from any parameter.
```

Here, the generic is performing the role of a type assertion. There is no guarantee that the runtime return value will be whatever is specified for `T`. It could be anything.

Instead the function should specify `unknown` as the return type, requiring the caller to use an explicit type assertion — or, better still, a [user-defined type guard](https://www.typescriptlang.org/docs/handbook/advanced-types.html#user-defined-type-guards).

There are exceptions to the rule — for example, some RxJS operators would fail, as they infer a type parameter from the source observable upon which `pipe` is called. However, exceptions are relatively rare and my recommendation would be to use the rule as a warning — or as an error with local overrides.

You can find the rule in [`eslint-plugin-etc`](https://github.com/cartant/eslint-plugin-etc). It has same name as the Wotan rule: [`no-misused-generics`](https://github.com/cartant/eslint-plugin-etc/blob/9a023f2b46adce9df086bb162bee06841bc50952/source/rules/no-misused-generics.ts).

The second rule has a related tweet, too. It's this one, from [Sophie Alpert](https://twitter.com/sophiebits):

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">WTB a lint rule that detects<br><br>const [processedData, setProcessedData] = useState();<br>useEffect(() =&gt; {<br> let processed = /* do something with data */;<br> setProcessedData(processed);<br>}, [data]);<br><br>and tells you to use useMemo instead.</p>&mdash; Sophie Alpert (@sophiebits) <a href="https://twitter.com/sophiebits/status/1293710971274289152?ref_src=twsrc%5Etfw">August 13, 2020</a></blockquote>

The problem with the code in the tweet's snippet is that it effects an additional render and reconciliation. The function passed to `useEffect` runs _after_ a render is committed and the `setState` call made within the function effects another render.

Wherever `useEffect` is used to memoize the synchronous processing of data, it can be replaced with `useMemo` and the additional render and reconciliation can be avoided, like this:

<!-- prettier-ignore -->
```js
const processedData = useMemo(() => {
  let processed = /* do something with data */;
  return processed;
}, [data]);
```

The lint rule that I wrote — [`prefer-usememo`](https://github.com/cartant/eslint-plugin-react-etc/blob/4a2a2a60d3e04076d647410ea5516c49a943cfb2/source/rules/prefer-usememo.ts) in [`eslint-plugin-react-etc`](https://github.com/cartant/eslint-plugin-react-etc) — uses a simple heuristic to determine whether or not a `useEffect` call can be replaced with `useMemo`.

The rule will suggest `useMemo` whenever the function passed to `useEffect`:

- makes an single, unconditional, clearly-synchronous call to a `useState` setter;
- does not reference or call additional `useState` setters;
- has some dependencies; and
- does not return a teardown function.

The heuristic identifies the `useEffect` usage in Sophie's tweet — and suggests `useMemo` as a replacement — and it hasn't effected any false positives in the code bases over which I've run the rule. The rule's tests include some of the [use cases](https://github.com/cartant/eslint-plugin-react-etc/blob/4a2a2a60d3e04076d647410ea5516c49a943cfb2/tests/rules/prefer-usememo.ts#L39-L174) for which the rule will not suggest `useMemo` as a replacement.

The rule is implemented in TypeScript — there are other React lint rules that I'll be adding to [`eslint-plugin-react-etc`](https://github.com/cartant/eslint-plugin-react-etc) and they will take advantage of information from the type system — but it doesn't rely upon TypeScript-specific AST nodes, so it will work just fine with ESLint configurations that do not use the TypeScript parser.
