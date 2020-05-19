---
title: "TIL: Destructuring Property Assignment"
description: How to assign destructured properties to variables
date: "2019-12-22T16:24:00+1000"
categories: []
keywords: []
ckTags: []
cardImage: "./title.jpeg"
---

![Typewriter parts](title.jpeg "Photo by Florian Klauer on Unsplash")

Until recently, I wasn't aware of the JavaScript syntax for destructuring property assignment.

I knew that I could destructure array elements and object properties in variable declarations, like this:

```js
const [vowel] = ["a", "e", "i", "o", "u"];
console.log(vowel); // a

const { name } = { name: "Alice" };
console.log(name); // Alice
```

And I knew that I could destructure an array and assign an element to a previously declared variable, like this:

```js
let vowel;
[vowel] = ["a", "e", "i", "o", "u"];
console.log(vowel); // a
```

But I didn't know how to destructure an object and assign a property to a previously declared variable. I tried this:

```js
let name;
{ name } = { name: "Alice" };
```

But this error was effected:

```text
SyntaxError: Unexpected token '='
```

The problem was that the braces surrounding the `name` variable were parsed as a block. To be parsed as [destructuring property assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment), the assignment expression needs to be surrounded by parentheses, like this:

```js
let name;
({ name } = { name: "Alice" });
console.log(name); // Alice
```

It's worth noting that if you rely upon [automatic semicolon insertion](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#Automatic_semicolon_insertion), you may need to precede the parentheses with a semicolon to prevent the assignment expression from being used to execute a function on the previous line.

For example, this usage:

<!-- prettier-ignore -->
```js
let name
console.log("assigning")
({ name } = { name: "Alice" })
```

Will effect this error:

```text
TypeError: console.log(...) is not a function
```
