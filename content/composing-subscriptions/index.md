---
title: "RxJS: Composing Subscriptions"
description: Coordinating unsubscription using composition
date: "2018-02-16T04:37:40.717Z"
categories: []
keywords: []
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-composing-subscriptions-b53ab22f1fd5
---

![Matryoshka dolls](title.jpeg "Photo by Bradley Davis on Flickr")

RxJS code involves making subscriptions to observables. Lots of subscriptions.

If each subscription is assigned to its own variable or property, the situation can be difficult to manage.

Fortunately, there are techniques — such as using `takeUntil` or `takeWhile` — that make dealing with subscriptions much easier. If you’re not familiar with the techniques, you should read [_Don’t Unsubscribe_](https://medium.com/@benlesh/rxjs-dont-unsubscribe-6753ed4fda87).

However, there are situations in which you still need to deal with subscriptions. For example, it’s sometimes necessary to manage subscriptions when writing operators.

Let’s look at subscriptions and at how subscription composition can be used to simplify some user-land operators.

## What is a subscription?

The [`Subscription`](https://github.com/ReactiveX/rxjs/blob/5.5.6/src/Subscription.ts) class and its related types look like this:

```ts
interface AnonymousSubscription {
  unsubscribe(): void;
}

type TeardownLogic = AnonymousSubscription | Function | void;

interface SubscriptionLike extends AnonymousSubscription {
  unsubscribe(): void;
  readonly closed: boolean;
}

class Subscription implements SubscriptionLike {
  closed: boolean = false;
  constructor(unsubscribe?: () => void) {
    /*...*/
  }
  unsubscribe(): void {
    /*...*/
  }
  add(teardown: TeardownLogic): Subscription {
    /*...*/
  }
  remove(subscription: Subscription) {
    /*...*/
  }
}
```

A `Subscription` instance is what’s returned from a call to `subscribe` and, most of the time, it’s only the subscription’s `unsubscribe` method that’s called. However, the `Subscription` class also has `add` and `remove` methods.

The `add` method can be used to add a child subscription — or a tear-down function — to a parent subscription. When a parent subscription is unsubscribed, any child subscriptions that were added to it are also unsubscribed.

The `remove` method removes a child subscription from a parent — something that you’re unlikely to need to do when composing subscriptions. However, the method does have a purpose: when a child unsubscribes, it removes itself from its parent.

Let’s have look to see how `add` can be used to simplify a user-land operator.

## Adding subscriptions

Here is an implementation of a pipeable operator that I call `subsequent`:

```ts
function subsequent<T>(
  count: number,
  operator: (source: Observable<T>) => Observable<T>
): (source: Observable<T>) => Observable<T> {
  return (source: Observable<T>) =>
    new Observable<T>((observer) => {
      const published = source.pipe(publish()) as ConnectableObservable<T>;
      const concatenated = concat(
        published.pipe(take(count)),
        published.pipe(operator)
      );
      const concatSubscription = concatenated.subscribe(observer);
      const connectSubscription = published.connect();
      return () => {
        concatSubscription.unsubscribe();
        connectSubscription.unsubscribe();
      };
    });
}
```

The `subsequent` operator mirrors the source and allows the first `count` notifications to flow through unchanged, but applies the specified `operator` to all subsequent notifications.

It works by using `publish` to obtain a hot observable that mirrors the source and then uses `concat` to ensure subscriptions to the initial and subsequent observables occur in a particular order.

There are two subscriptions: one is made to the `concatenated` observable; and another is made to the `published` observable — via the call to `connect`.

The tear-down function returned — from the function passed to `Observable.create` — needs to call `unsubscribe` on both subscriptions.

This can be made a little simpler using subscription composition:

```ts
function subsequent<T>(
  count: number,
  operator: (source: Observable<T>) => Observable<T>
): (source: Observable<T>) => Observable<T> {
  return (source: Observable<T>) =>
    new Observable<T>((observer) => {
      const published = source.pipe(publish()) as ConnectableObservable<T>;
      const concatenated = concat(
        published.pipe(take(count)),
        published.pipe(operator)
      );
      const subscription = concatenated.subscribe(observer);
      subscription.add(published.connect());
      return subscription;
    });
}
```

Instead of using a variable to keep track of the subscription made to the `published` observable, it can be added to the subscription made to the `concatenated` observable.

And instead of returning a tear-down function, the function passed to `Observable.create` can return the subscription itself. Then, when a subscriber unsubscribes from the resultant observable, `unsubscribe` will be called on the subscription and the subscriptions to both the `concatenated` and `published` observables will be unsubscribed.

## Creating explicit subscriptions

Subscriptions can be composed using a `Subscription` instance returned from a call to `unsubscribe`, but they can also be composed using explicitly-created subscriptions.

Here is an implementation of a pipeable operator that I call `prioritize`:

```ts
function prioritize<T, R>(
  selector: (
    prioritized: Observable<T>,
    deprioritized: Observable<T>
  ) => Observable<R>
): (source: Observable<T>) => Observable<R> {
  return (source: Observable<T>) =>
    new Observable<T>((observer) => {
      const published = publish<T>()(source) as ConnectableObservable<T>;
      const prioritized = new Subject<T>();
      const subscription = new Subscription();
      subscription.add(published.subscribe(prioritized));
      subscription.add(selector(prioritized, published).subscribe(observer));
      subscription.add(published.connect());
      return subscription;
    });
}
```

It calls a `selector`, passing two observables that mirror the source. However, the first observable is guaranteed to have been subscribed to the source first — so notifications from the first observable will occur before those from the second. This is sometimes useful when composing the notification observables needed by operators like `bufferWhen`.

There are several subscriptions that occur in this implementation and instead of adding to a subscription returned from a call to `subscribe`, the implementation creates an explicit parent `Subscription` and adds the child subscriptions to it.

## Gotchas

Something to be aware of, when composing subscriptions, is that the unsubscription order can be important.

For example, if `prioritze` were to have this simpler implementation, it would have a bug:

```ts
const published = publish<T>()(source) as ConnectableObservable<T>;
const prioritized = new Subject<T>();
const subscription = published.subscribe(prioritized);
subscription.add(selector(prioritized, published).subscribe(observer));
subscription.add(published.connect());
return subscription;
```

Here, the subscription returned is the subscription made when the `prioritized` subject is subscribed to the `published` observable — not the subscription made to the observable to which the `observer` is subscribed.

The introduced bug is this:

- When the source completes, the completion notification is first received by the `prioritized` subject, which then unsubscribes.
- When the `prioritized` subject unsubscribes, it also unsubscribes the subscription to observable returned by the `selector`.
- The observable returned by the `selector` will not have received the complete notification, so the `observer` will not have received it either. That means the complete notification from the source will not have been mirrored.

As a general rule, if you have several subscriptions in a user-land operator and are using subscription composition, you should use an explicit, parent `Subscription` instance.
