---
title: "RxJS: multicast’s Secret"
description: "Multicasting without a connectable observable"
date: "2017-08-19T12:55:19.002Z"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title.jpeg"
slug: "/@cartant/rxjs-multicasts-secret-760e1a2b176e"
---

![Test pattern](title.jpeg "Photo by Tim Mossholder on Unsplash")

`multicast` has a secret. And so does `publish` — which wraps `multicast`. And it’s sometimes really useful.

## The secret

The documentation for `multicast` and `publish` mentions a `ConnectableObservable`. A connectable observable is type of observable that waits until its `connect` method is called before it begins emitting notifications to subscribers. However, the `multicast` and `publish` operators don’t always return a connectable observable.

Let’s start by looking at the [source code](https://github.com/ReactiveX/rxjs/blob/5.4.3/src/operator/publish.ts#L24-L27) for `publish`:

```ts
export function publish<T>(
  this: Observable<T>,
  selector?: (source: Observable<T>) => Observable<T>
): Observable<T> | ConnectableObservable<T> {
  return selector
    ? multicast.call(this, () => new Subject<T>(), selector)
    : multicast.call(this, new Subject<T>());
}
```

It’s clear that `publish` is just a thin wrapper around `multicast`. It creates a subject, which is passed to `multicast` along with the optional `selector` function. The interesting bit is inside the `multicast` implementation, which contains the [following code](https://github.com/ReactiveX/rxjs/blob/5.4.3/src/operator/multicast.ts#L42-L50):

```ts
if (typeof selector === "function") {
  return this.lift(new MulticastOperator(subjectFactory, selector));
}

const connectable: any = Object.create(this, connectableObservableDescriptor);
connectable.source = this;
connectable.subjectFactory = subjectFactory;
return <ConnectableObservable<T>>connectable;
```

`multicast` will only return a connectable observable if a selector function is **not** specified. If a function is specified, the [lift mechanism](https://github.com/ReactiveX/rxjs/issues/60) will be used to allow the source observable to create an observable of the appropriate type. There will be no need to call `connect` on the returned observable and the source observable will be shared within the scope of the selector function.

That means that `multicast` (and `publish`) can be used to easily implement the local sharing of a source observable.

## Local sharing with publish

Let’s look at an example that uses `publish`.

RxJS includes a `defaultIfEmpty` operator which takes a value that’s to be emitted if the source observable is empty. Sometimes, it’s useful be be able to specify a default observable — rather than a single value — so let’s implement a `defaultObservableIfEmpty` function that can be used with the `let` operator.

A marble diagram, showing its behaviour with an empty source, looks like this:

![](diagram-1.png)

RxJS includes an `isEmpty` operator which will emit a boolean value when the source observable completes — indicating whether or not the source was empty. However, for it to be used it in the implementation of `defaultObservableIfEmpty`, the source observable needs to be shared — as the value notifications need to be emitted, too, and `isEmpty` does not do that. `publish` makes sharing the source observable easy and the implementation looks like this:

```ts
function defaultObservableIfEmpty<T>(
  defaultObservable: Observable<T>
): (source: Observable<T>) => Observable<T> {
  return (source) =>
    source.publish((shared) =>
      shared.merge(
        shared
          .isEmpty()
          .mergeMap((empty) =>
            empty ? defaultObservable : Observable.empty<T>()
          )
      )
    );
}
```

`publish` is passed a selector function that receives the shared source observable. The `selector` returns an observable composed from the shared source, merged with either the default observable, if the source is empty, or with an empty observable, if the source is not empty.

The sharing of the source observable is managed entirely by `publish`. Within the selector, it’s possible to subscribe to the shared observable as many times as is necessary, without effecting further subscriptions to the source.

## Local sharing with multicast

Let’s look at another example, this time using `multicast`.

RxJS includes a `takeWhile` operator which returns an observable that emits values received from the source until a received value fails the predicate, at which point the observable completes. The value that fails the predicate is not emitted. Let’s implement a `takeWhileInclusive` function that can be used with the `let` operator.

A marble diagram, showing its behaviour with a value that fails the predicate, looks like this:

![](diagram-2.png)

The implementation can use the `takeWhile` operator as its basis; it just needs to concatenate the last value, if it fails the predicate. To obtain the last value — after the observable returned by `takeWhile` completes — a `ReplaySubject` can be used:

```ts
function takeWhileInclusive<T>(
  predicate: (value: T) => boolean
): (source: Observable<T>) => Observable<T> {
  return (source) =>
    source.multicast(
      () => new ReplaySubject<T>(1),
      (shared) =>
        shared
          .takeWhile(predicate)
          .concat(shared.take(1).filter((t) => !predicate(t)))
    );
}
```

The source observable is shared using a `ReplaySubject` with a buffer size of one. When the observable returned by the `takeWhile` operator completes, the shared observable is concatenated, with `take(1)` ensuring that only the replayed value is considered and `filter` ensuring that it’s only appended if it fails the predicate.

## Can this behaviour be relied upon?

RxJS version 5 is a relatively new library and its documentation is a work in progress, so the behaviour is not-yet-documented, rather than internal. The public [TypeScript signatures](https://github.com/ReactiveX/rxjs/blob/5.4.3/src/operator/multicast.ts#L7-L10) indicate that a `ConnectableObservable` is not always returned and there are [unit tests](https://github.com/ReactiveX/rxjs/blob/5.4.3/spec/operators/multicast-spec.ts#L86-L144) for the behaviour.

There are other ways to implement these functions — which is often the case with RxJS — but the above examples show that when the local sharing of a source observable is necessary, `publish` and `multicast` are easy to use and are worth considering.
