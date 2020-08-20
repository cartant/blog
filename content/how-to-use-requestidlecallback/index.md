---
title: "RxJS: How to Use request​Idle​Callback"
description: "Thoughts on how request​Idle​Callback could be used"
date: "2020-06-26T13:10:00+1000"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title-card.png"
---

![hourglass](title.png)

At least three times in the last few weeks, I've been asked about whether or not it would be possible — or whether there are plans — to write a scheduler that's based on [`requestIdleCallback`](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback), so I figured I ought to write a blog article about it.

The MDN documentation for `requestIdleCallback` has — at the top of the page — this warning:

> This is an experimental technology.  
> Check the [Browser compatibility table](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback#Browser_compatibility) carefully before using this in production.

The compatibility table in the MDN documentation — and on [caniuse.com](https://caniuse.com/#feat=requestidlecallback), too — shows that `requestIdleCallback` is not supported on Safari or iOS Safari. The table includes a link to the [draft specification](https://w3c.github.io/requestidlecallback/) — dated 30 December 2019.

From that, the second part of the question can be answered: no, there is no chance that support for `requestIdleCallback` will be added to the RxJS core until the feature is widely supported. Adding support for a feature that's unavailable on both Safari and iOS Safari is not something that I would favour and I would expect it would be something that other core team members would be unlikely to entertain.

The first part of the question is a little more interesting.

I think it's not so much a question of whether a scheduler based on `requestIdleCallback` _could_ be written. Rather, it's _should_ such a scheduler be written?

I think the answer is _no_.

## How would it work?

If a scheduler based on `requestIdleCallback` were to be written, how would it work?

In the RxJS core, the `queueScheduler` and `asapScheduler` implementations fall back to the `asyncScheduler` whenever a duration is specified. This makes sense, as both the `queueScheduler` and `asapScheduler` are 'faster' than the `asyncScheduler` — the former is synchronous and the latter is a micro task, whereas the `asyncScheduler` is a macro task.

It makes less sense for an idle scheduler.

If an idle scheduler where to be called with a duration of zero, its scheduled action might not be executed for a considerable period of time — i.e. not until the browser is idle. However, if it were called with a duration of one millisecond, falling back to the `asyncScheduler` would schedule the action for execution for the next millisecond — regardless of whether the browser is idle at that time or not.

Falling back from the `animationFrameScheduler` to the `asyncScheduler` is arguably weird behaviour, too. Which is one of the reasons there is now an [`animationFrames`](https://github.com/ReactiveX/rxjs/blob/96868ac754c0147a9aa61182185f27224eb7f11a/src/internal/observable/dom/animationFrames.ts) observable in the version 7 beta of RxJS.

There is also the question of what an idle scheduler should do if an action is scheduled when an action is already pending — i.e. when the scheduler is waiting for the browser to become idle so that an already scheduled action can be executed. Should it execute both actions when the browser becomes idle? Should it execute one action and the wait for the browser to again become idle before executing the second?

## Using an idle observable instead

Given that it's questionable to fallback to the `asyncScheduler` and that it's not clear how an idle scheduler should schedule its actions, let's look at whether or not an idle observable could be used instead.

Let's implement an `idle` observable like this:

```ts
import { Observable } from "rxjs";

export function idle(): Observable<void> {
  return new Observable<void>((observer) => {
    const handle = requestIdleCallback(() => {
      observer.next();
      observer.complete();
    });
    return () => cancelIdleCallback(handle);
  });
}
```

Unlike the `animationFrames` observable, our `idle` observable emits only once — when the browser becomes idle — and then completes. It does this so that it doesn't continuously emit notifications whenever the browser happens to be idle.

Let's see how we could compose observables using `idle`.

If we have a source that emits values upon which some work needs to performed, we can to this:

<!-- prettier-ignore -->
```ts
source.pipe(
  map((value) => work(value))
);
```

Here, the work will be performed whenever the source emits a `next` notification.

If we want to wait until the browser is idle before performing the work, we can use the `idle` observable, like this:

<!-- prettier-ignore -->
```ts
source.pipe(
  bufferWhen(() => idle()),
  mergeMap((buffer) => buffer.map(work))
);
```

Here, our composed observable will wait until the browser is idle before performing the work on each buffered value.

If we want to perform the work on only a single value each time the browser becomes idle, we could do this:

<!-- prettier-ignore -->
```ts
source.pipe(
  concatMap((value) => idle().pipe(
    map(() => work(value))
  ))
);
```

Here, the composed observable will perform the work on one value and will then wait until the browser again becomes idle before performing the next.

To me, this seems pretty flexible, so this is the approach that I'd be inclined to take when/if `requestIdleCallback` becomes more widely supported.
