---
title: "RxJS: How to Write a delayUntil Operator"
description: A step-by-step breakdown of writing an operator that uses a notifier
date: "2020-02-24T19:05:00+1000"
categories: []
keywords: []
cardImage: "./title-card.jpeg"
---

![Traffic light](title.jpeg "Photo by bady qb on Unsplash")

Recently, I was asked — [on Twitter](https://twitter.com/OliverJAsh/status/1230975053929570306) — if there is a built-in operator that will delay notifications from a source until a signal is received from a notifier.

There isn't, but it's possible to write one and that's what this article is about. Let's do it step-by-step, starting with the signature.

The operator is only going to delay the notifications that it receives from the source. It isn't going to manipulate the values within the notifications, so the observable's element type is not going to change. Knowing that, we can start by writing the operator's signature like this:

```ts
import { OperatorFunction } from "rxjs";

function delayUntil<T>(notifier: /* todo */): OperatorFunction<T, T> {
  /* todo */
}
```

As with [`takeUntil`](https://github.com/ReactiveX/rxjs/blob/5cf26e7f567dcc482d37a447eaa61c00d4786f46/src/internal/operators/takeUntil.ts#L51), we'll use an observable as the notifier. The operator doesn't care about the value that it receives from the notifier — only that has received one — so we should use `Observable<any>` as the type:

```ts
import { Observable, OperatorFunction } from "rxjs";

function delayUntil<T>(notifier: Observable<any>): OperatorFunction<T, T> {
  /* todo */
}
```

The operator's implementation is going to split the source notifications into two streams: one stream for the pre-signal behaviour; and another for the post-signal behaviour. That means it's going to need to subscribe to the source observable twice.

The implementation cannot subscribe directly to the source more than once — for example, with HTTP sources, multiple subscriptions would effect multiple requests — so the source has to be shared.

We can do that using [the `publish` operator and a selector](/calling-publish-with-a-selector):

<!-- prettier-ignore -->
```ts
import { Observable, OperatorFunction } from "rxjs";
import { publish } from "rxjs/operators";

function delayUntil<T>(notifier: Observable<any>): OperatorFunction<T, T> {
  return source => source.pipe(
    publish(published => /* todo */)
  );
}
```

Inside the selector, we can subscribe to the `published` observable as many times as we like without effecting multiple subscriptions to the `source`.

After the source notifications are split into two streams, they will need to be joined together. We know that the first stream — the pre-signal, delayed notifications — ends when the signal is received and that the second stream — the post-signal notifications — just mirrors the source, so we can use `concat` to join the streams and we can use the `published` observable itself for the second stream:

```ts
import { concat, Observable, OperatorFunction } from "rxjs";
import { publish } from "rxjs/operators";

function delayUntil<T>(notifier: Observable<any>): OperatorFunction<T, T> {
  return source =>
    source.pipe(
      publish(published =>
        concat(
          /* todo */,
          published
        )
      )
    );
}
```

Now we need to figure out how to implement the stream of delayed notifications.

We know that it's going to be based on the source notifications, so we can start with:

<!-- prettier-ignore -->
```ts
published.pipe(/* todo */)
```

We want to accumulate values until the notifier's signal is received. The [`buffer`](https://rxjs.dev/api/operators/buffer) operator can do that. It buffers values received from its source until it receives the notifier's signal. It then emits those values in an array and resumes buffering the source:

<!-- prettier-ignore -->
```ts
published.pipe(buffer(notifier), /* todo */)
```

That gets us part of the way there, but we don't want it to resume buffering once the signal is received. We want the delayed stream to complete, so that `concat` subscribes to its second argument — the `published` observable — and starts mirroring the source.

We can get the behaviour we want using the [`take`](https://rxjs.dev/api/operators/take) operator:

<!-- prettier-ignore -->
```ts
published.pipe(buffer(notifier), take(1), /* todo */)
```

The delayed stream now buffers source values until a signal is received from the notifier, at which time it emits an array of values and then completes. But we don't want an array of values emitted into the stream; we want the values themselves emitted. That means we need to flatten the stream and we can use the [`mergeMap`](https://rxjs.dev/api/operators/mergeMap) operator to do that:

<!-- prettier-ignore -->
```ts
published.pipe(buffer(notifier), take(1), mergeMap(values => values))
```

We can do this because an array is a valid [`ObservableInput`](https://github.com/ReactiveX/rxjs/blob/5cf26e7f567dcc482d37a447eaa61c00d4786f46/src/internal/types.ts#L52) and when treated as an observable source, an array's elements are emitted into the stream.

We can make this a little less verbose using the [`mergeAll`](https://rxjs.dev/api/operators/mergeAll) operator. When applied to a higher-order observable — an observable whose elements are themselves valid observable inputs — `mergeAll` flattens each element it receives:

<!-- prettier-ignore -->
```ts
published.pipe(buffer(notifier), take(1), mergeAll())
```

Finally, after adding our delayed stream to the `concat` call, the operator is finished:

<!-- prettier-ignore -->
```ts
import { concat, Observable, OperatorFunction } from "rxjs";
import { buffer, mergeAll, publish, take } from "rxjs/operators";

function delayUntil<T>(notifier: Observable<any>): OperatorFunction<T, T> {
  return source =>
    source.pipe(
      publish(published =>
        concat(
          published.pipe(buffer(notifier), take(1), mergeAll()),
          published
        )
      )
    );
}
```

Of course, it's a good idea to write [some tests](https://github.com/cartant/rxjs-etc/blob/f7fbc18119850fd35b9755b727a3f1d93df83fdf/source/operators/delayUntil-spec.ts) to ensure that the operator behaves as expected, too.
