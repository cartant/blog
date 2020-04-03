---
title: "RxJS: Understanding Expand"
description: A look at the expand operator
date: "2018-02-19T22:38:45.159Z"
categories: []
keywords: []
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-understanding-expand-a5f8b41a3602
---

![Effect pedals](title.jpeg "Photo by mhx on Flickr")

RxJS has a lot of operators. Lots and lots of them.

It takes time to learn what they all do and how they can be used. Some operators are straightforward; others, less so. One operator that developers often find confusing is `expand`.

It doesn’t have to be confusing, though. It’s just like a delay pedal.

## So what’s a delay pedal?

A delay pedal is an electronic effect that simulates echoes. You plug an instrument into it, plug it into the amp, stomp on it to switch it on and then cool things happen when you make some noise.

The input signal received by the pedal is passed straight through, as the point of the effect is to add echoes — not to modify the input signal itself. The pedal then records the signal and replays it after a delay — simulating an echo.

A typical delay pedal will have a feedback control and turning it up will see the delayed signal fed back into the effect, as if it came from the source. So a delay will be applied to the delayed signal, effecting a series of ever-quieter echoes.

## So how’s expand like a pedal?

The key to understanding how `expand` works is realising that — just like a pedal — the signal that’s received from the source passes straight through. That is, the `expand` operator mirrors the source observable’s notifications.

Additionally — like a pedal — the operator performs some processing on the notifications: it passes them to the supplied `project` function.

The observables returned from the `project` function are merged with the source observable — again, like a pedal — with the merge occurring _before_ the operator. That is, the notifications from the returned observables are treated as if they were notifications from the source.

## So how can expand be used?

One situation to which the `expand` operator is well-suited is paging. Let’s have a look at how it could be used to page through a list of GitHub repositories.

When a [request](https://developer.github.com/v3/repos/#list-user-repositories) is made for a GitHub resource like this:

```text
GET /users/sindresorhus/repos
```

A [response](https://developer.github.com/v3/repos/#response) representing the first page of results is received. The response has JSON content — an array of repo entities — and a `Link` header that looks something like this:

```text
Link: <https://api.github.com/user/170270/repos?page=2>; rel="next",
      <https://api.github.com/user/170270/repos?page=33>; rel="last"
```

The `next` link is the URL for the next page of results and it’s this that makes `expand` well-suited to paging: each page knows what needs to be requested to obtain the next page.

Let’s use RxJS’s `ajax` observable to implement a `get` function:

```ts
import { Observable } from "rxjs/Observable";
import { ajax } from "rxjs/observable/dom/ajax";
import { AjaxResponse } from "rxjs/observable/dom/AjaxObservable";
import { map } from "rxjs/operators";

export function get(
  url: string
): Observable<{
  content: object[];
  next: string | null;
}> {
  return ajax.get(url).pipe(
    map((response) => ({
      content: response.response,
      next: next(response)
    }))
  );
}

function next(response: AjaxResponse): string | null {
  let url: string | null = null;
  const link = response.xhr.getResponseHeader("Link");
  if (link) {
    const match = link.match(/<([^>]+)>;\s*rel="next"/);
    if (match) {
      [, url] = match;
    }
  }
  return url;
}
```

The `get` function returns an observable that emits an object containing a page’s content and the URL for the next page (or `null`, if it’s the last page).

With the `get` function, paging using the `expand` operator is simple:

```ts
import { empty } from "rxjs/observable/empty";
import { concatMap, expand } from "rxjs/operators";
import { get } from "./get";

const url = "https://api.github.com/users/sindresorhus/repos";
const repos = get(url).pipe(
  expand(({ next }) => (next ? get(next) : empty())),
  concatMap(({ content }) => content)
);
repos.subscribe((repo) => console.log(repo));
```

The observable returned from the initial call to the `get` function emits only a single value: the first page.

This value is received by the `expand` operator and is emitted — mirroring the source observable. It’s also passed to the `project` function, which checks to see if there is a `next` page. If there is, a call to `get` for the next page feeds back into the observable stream, as if its notifications were from the source — just like an echo feeds back with a delay pedal.

If there is no next page, the `project` function returns an `empty` observable and the feedback stops.

The `concatMap` operator flattens the array of repos into the stream — so that the observable emits individual repo entities, rather than arrays of repo entities.

Simple, eh? Rock ’n’ roll, people.
