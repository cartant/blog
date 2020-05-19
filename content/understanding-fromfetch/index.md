---
title: "RxJS: Understanding fromFetch"
description: A look at how fromFetch uses fetch and AbortController
date: "2020-05-19T17:17:00+1000"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title-card.jpeg"
---

![Short-coated puppy biting rope](title.jpeg "Photo by Brendan Hollis on Unsplash")

Since version 5, RxJS has included an `ajax` observable — a framework-independent [XHR](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) observable for HTTP requests.

It has always been possible to compose observables using `fetch` — as promises [play nice](https://medium.com/@benlesh/rxjs-observable-interop-with-promises-and-async-await-bebb05306875) with RxJS — but the advantage in using `ajax` was that an HTTP request could be aborted when an observable's subscriber unsubscribed.

For a long time, it was not possible to abort HTTP requests initiated using `fetch`. These days, that's no longer the case, as browser support for aborting `fetch`-initiated requests is widespread.

To take advantage of this, RxJS has a new API: `fromFetch` — which was introduced in version 6.5.0.

## Some history

Being promise-based, `fetch`'s support for aborting ongoing requests initially depended upon the [TC39 proposal for cancellable promises](https://github.com/tc39/proposal-cancelable-promises), but that proposal was withdrawn in 2016.

In 2017, a mechanism for aborting an ongoing `fetch` was added to the DOM Standard and [Jake Archibald](https://twitter.com/jaffathecake) covers it in detail in [this article](https://developers.google.com/web/updates/2017/09/abortable-fetch):

> Our alternative proposal, `AbortController`, didn't require any new syntax, so it didn't make sense to spec it within TC39. Everything we needed from JavaScript was already there, so we defined the interfaces within the web platform, specifically [the DOM standard](https://dom.spec.whatwg.org/#aborting-ongoing-activities). Once we'd made that decision, the rest came together relatively quickly.

It took some time for support for this mechanism to become widespread — Safari didn't implement `AbortController` until version 12.1, which was released in March 2019.

## AbortController

The mechanism for aborting ongoing `fetch` requests looks like this:

```ts
const controller = new AbortController();
const { signal } = controller;

setTimeout(() => controller.abort(), 5e3);

fetch(url, { signal })
  .then((response) => response.text())
  .then((text) => console.log(text));
```

It uses an `AbortController` to signal when a `fetch` request is to be aborted. The `signal` is passed via the `fetch` call's `RequestInit` parameter and, internally, `fetch` calls `addEventListener` on the `signal` — listening for the the `"abort"` event.

If the `signal` emits an `"abort"` event whilst the request is ongoing, the promise returned by `fetch` rejects with an `AbortError`.

In the above snippet, the request will be aborted if it takes longer than five seconds to be fulfilled. If it's fulfilled within five seconds, calling `abort` on the `controller` is ineffectual.

This RxJS equivalent — that uses `fromFetch` — looks like this:

```ts
import { timer } from "rxjs";
import { fromFetch } from "rxjs/fetch";
import { mergeMap, takeUntil } from "rxjs/operators";

fromFetch(url)
  .pipe(
    mergeMap((response) => response.text()),
    takeUntil(timer(5e3))
  )
  .subscribe((text) => console.log(text));
```

`fromFetch` accepts the same parameters as `fetch` and, internally, it creates an `AbortController`. The controller is wired up to the returned `Subscription`. When the subscriber unsubscribes, the controller is signaled and, if it's ongoing, the request is be aborted.

The advantage of using `fromFetch` isn't that less code is required — there is a similar amount in each of the above snippets — it's that the `fromFetch` is composable.

For example, `fromFetch` can be used within `switchMap`, like this:

<!-- prettier-ignore -->
```ts
switchMap((url) =>
  fromFetch(url).pipe(mergeMap((response) => response.text()))
)
```

Here, requests will be aborted if the source observable emits another value before said request is fulfilled.

## Chunked responses

You might have noticed that there are two promises involved in the `fetch` example above:

- the promise returned by `fetch`; and
- the promise returned by the response's `text` method.

The reason for this is that the promise returned by `fetch` resolves as soon as the response's headers are available, but the promise returned by the response's `text` method — or its `json` method, etc. — does not resolve until the entire response has been received.

In HTTP 1.1, responses can use the [chunked transfer encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding):

> Chunked encoding is useful when larger amounts of data are sent to the client and the total size of the response may not be known until the request has been fully processed.

Using that encoding, responses are split into chunks — with the first chunk containing the headers, like this:

![Chunked response](chunked-response-widened.png)

That means a significant amount of time can elapse between the first and second promises resolvingc.

The `AbortController` mechanism deals with this by aborting both promises when it signals. That means that if the controller signals after the promise returned by `fetch` has resolved — and the caller has received a `Response` — but before the promise returned by the `text` method has resolved, the `text`-method-returned promise will reject with an `AbortError`.

The observable returned by `fromFetch` emits a `Response` and it's emitted as soon as the response's headers are received. Immediately after the `Response` is emitted, the observable completes and its subscriber is automatically unsubscribed. And that means the emitted `Response` could still be ongoing.

To support aborting chunked responses — upon unsubscription — `fromFetch` accepts a `selector` via the `RequestInit` argument, like this:

<!-- prettier-ignore -->
```ts
switchMap((url) =>
  fromFetch(url, {
    selector: (response) => response.text(),
  })
)
```

If a `selector` is passed, both promises remain controlled by the internal `AbortController` and if unsubscription occurs after the headers are received, but before the complete body is received, the request will be aborted.

And one last thing: if you are going to use `fromFetch` in conjunction with `switchMap` to abort ongoing requests, [make sure it's safe to do that](/avoiding-switchmap-related-bugs/).
