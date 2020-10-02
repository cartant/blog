---
title: "Conventional Comments"
description: "Are conventional comments useful in code reviews?"
date: "2020-10-02T16:59:00+1000"
categories: []
keywords: []
ckTags: []
cardImage: "./title-card.jpeg"
---

![Typewriter](title.jpeg "Photo by Florian Klauer on Unsplash")

I've done a lot of code reviews, over the last month or so.

A _lot_ of code reviews.

[Ben](https://twitter.com/BenLesh) has been refactoring — well, rewriting, really — almost the entire RxJS codebase, so my inbox has seen a steady stream of review requests. Since the middle of August, [about 60 of these refactor/rewrite pull requests](https://github.com/ReactiveX/rxjs/pulls?q=is%3Apr+author%3Abenlesh+updated%3A%3E2020-08-10) have been opened. And some of them have been [whoppers](https://github.com/ReactiveX/rxjs/pull/5729).

Coincidentally, in early August, I read about [conventional comments](https://conventionalcomments.org/) and decided to try them out in my code reviews.

The gist is that a comment is prefixed with a label: _praise_, _nitpick_, _suggestion_, _issue_, _question_, _thought_, or *chore* — much like the descriptions in [conventional commits](https://www.conventionalcommits.org). The point of the label is to more clearly convey the intention of the comment and to set the the tone.

For example, a comment like this isn't particularly helpful:

```text
This is not worded correctly.
```

Whereas, with a label, the author has a better idea what action the reviewer thinks it necessary (maybe none):

```text
suggestion: This is not worded correctly.
```

or:

```text
nitpick (non-blocking): This is not worded correctly.
```

I think conventional comments have improved my reviews.

The reality is that open-source contributions aren't usually a developer's highest priority. My reviews are often done in between other tasks or late at night, after other tasks have been wrapped up. I know that I have been guilty of writing comments that are too terse or too open to interpretation. Having to choose a label for the comment makes me think more about what I'm saying — e.g. is it a _nitpick_, a _question_ or a _suggestion_?

What I like most is that if I mark a review as _changes requested_, it's clear to the author what it is that I think needs to be changed: the comments marked with _issue_ labels. The author can address — or debate — the _issue_ comments and can then decide whether or not any _nitpick_, _suggestion_ or _thought_ comments warrant changes.

It also means that I can leave a bunch of _nitpick_, _suggestion_ or _thought_ comments and approve the PR, allowing author to address the comments — if warranted — and then do the merge. It's a more efficient process — than re-reviewing — when developers are in separate time zones.

So are conventional comments worth using in code reviews?

It depends. 😅

I think they're useful in highly-asynchronous situations in which a miscommunication can take hours or days to resolve, so I'm going to keep using them for open-source reviews. YMMV.
