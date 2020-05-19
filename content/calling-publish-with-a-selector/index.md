---
title: "RxJS: Calling publish with a Selector"
description: How to replace connectable observables with selectors
date: "2019-04-02T09:45:50.204Z"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-calling-publish-with-a-selector-3ab48f052a4b
---

![Stacked books](title.jpeg "Photo by Beatriz Pérez Moya on Unsplash")

The `publish` and `publishReplay` operators can be called in two ways: either with or without a selector function. Let’s look at the differences and at why you should almost always pass a selector.

## Without a selector

When `publish` is called without a selector, it returns a particular type of observable: a `ConnectableObservable`.

A `ConnectableObservable` is a little unusual. It does not connect — i.e. subscribe — to its source when a subscriber subscribes. Instead, a `ConnectableObservable` subscribes to its source when its `connect` method is called.

This behaviour can be used to control the order in which subscriptions occur and to compose observables before a subscription is made to the source.

Let’s look at an example.

I’ve seen a `ConnectableObservable` used in a Stack Overflow answer on several occasions, so let’s make up a question and then answer it:

> How can I compose an observable so that a specified, initial value is emitted if the source observable does not emit a value within a specified duration?

Here’s a solution that uses `race`:

```ts
import { concat, ConnectableObservable, of, race, timer } from "rxjs";
import { delay, mapTo, publish } from "rxjs/operators";

const source = of(42).pipe(delay(100));
const published = source.pipe(publish()) as ConnectableObservable<number>;
const composed = race(published, concat(timer(10).pipe(mapTo(54)), published));
composed.subscribe((value) => console.log(value));
published.connect();
```

The solution uses `publish` to obtain a `ConnectableObservable` that’s then passed to `race`:

- by itself, as the first argument, to mirror the source; and
- concatenated to a `timer` — that waits for the specified duration before emitting the initial value — as the second argument.

The race will be won by whichever argument’s observable is the first to emit and `race` then mirrors that observable:

- if the source emits within the duration, it wins and the `timer` will be unsubscribed; otherwise
- if the `timer` emits, it wins and any subsequent notifications from the source will be concatenated.

The solution calls `connect` _after_ the subscription is made to the composed observable. It’s important that is does this, as doing so ensures that the solution will work for all sources. Calling `connect` _before_ `subscribe`, would not work with a synchronous source — a synchronous source’s notifications would be received by the `publish` implementation before the `ConnectableObservable` had subscribers and those notifications would be lost.

The solution is typical of a Stack Overflow answer: it demonstrates how the problem can be solved, but it does so in an isolated context.

In an application context, applying the solution gets complicated. The subscriber to has to do two things: call `subscribe`; and then call `connect`. And the order of those calls is important.

The solution could be made more composable by bundling the `subscribe`/`connect` logic into a `new Observable`, but that would be a little complicated.

Fortunately, there is a another way — a better way — of solving the problem: we can pass a selector.

## With a selector

When called with a selector, `publish` does not return a `ConnectableObservable` and there is no need to call `connect`.

The solution is simpler and looks like this:

```ts
import { concat, of, race, timer } from "rxjs";
import { delay, mapTo, publish } from "rxjs/operators";

const source = of(42).pipe(delay(100));
const composed = source.pipe(
  publish((published) =>
    race(published, concat(timer(10).pipe(mapTo(54)), published))
  )
);
composed.subscribe((value) => console.log(value));
```

When a selector is specified, the implementation — in [`multicast`](https://github.com/ReactiveX/rxjs/blob/6.4.0/src/internal/operators/multicast.ts#L58-L69) — uses the same mechanism as the initial solution, but does so without a `ConnectableObservable`:

```ts
call(subscriber: Subscriber<R>, source: Observable<any>) {
  const { selector } = this;
  const subject = this.subjectFactory();
  const subscription = selector(subject).subscribe(subscriber);
  subscription.add(source.subscribe(subject));
  return subscription;
}
```

Let’s compare the above snippet with the initial, `ConnectableObservable`\-based solution:

- When a subscriber subscribes, a subject is created and — like the `ConnectableObservable` — it forms the basis of the composed observable.
- An observable is then composed by passing the subject to the selector function and — as is the case in the initial solution — that composed observable can subscribe to the subject multiple times without effecting multiple subscriptions to the source.
- The subscriber is then subscribed to the observable returned by the selector function. At this stage, the subscriber will not receive notifications, as the subject has not been subscribed to a source — just as subscriptions to a `ConnectableObservable` will not receive notifications until it is connected.
- Finally, the subject is subscribed to the source and notifications flow through the composed observable to the subscriber. In the initial solution, this is what happens when `connect` is called.

Passing a selector effects the same outcome as using a `ConnectableObservable`, but it has several advantages:

- it’s simpler;
- it’s declarative;
- the caller does not have to call both `subscribe` and `connect`;
- the caller doesn’t have to deal with two subscriptions — both `subscribe` and `connect` return subscriptions; and
- it’s easy to compose further observables from the observable returned by `publish`.

To illustrate how easy it is to compose further observables from the selector-based solution, let’s create an operator.

The solution takes an observable and returns an observable, so all that needs to be done is to put it inside a function, like this:

```ts
import { concat, OperatorFunction, race, SchedulerLike, timer } from "rxjs";
import { mapTo, publish } from "rxjs/operators";

function startWithTimeout<T, S = T>(
  value: S,
  duration: number | Date,
  scheduler?: SchedulerLike
): OperatorFunction<T, S | T> {
  return (source) =>
    source.pipe(
      publish((published) =>
        race(
          published,
          concat(timer(duration, scheduler).pipe(mapTo(value)), published)
        )
      )
    );
}
```

## Always with a selector

Situations in which a `ConnectableObservable` needs to be explicitly manipulated by calling `connect` are rare — although, we will look at one such situation in a future article.

It’s almost always the case that a selector should be passed instead.

And that means — yep, you guessed it — we can use a linting rule to keep unnecessary `ConnectableObservable` manipulation out of our codebases.

I’ve added an `rxjs-no-connectable` rule to the [`rxjs-tslint-rules`](https://github.com/cartant/rxjs-tslint-rules) package. If you switch it on, it’ll remind you that you should almost always pass a selector.
