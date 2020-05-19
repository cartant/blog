---
title: "RxJS: Closed Subjects"
description: A look at subject unsubscription
date: "2018-02-06T11:42:09.818Z"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title.jpeg"
slug: /@cartant/rxjs-closed-subjects-1b6f76c1b63c
---

![Closed sign](title.jpeg "Photo by Tim Mossholder on Unsplash")

This article looks at the `unsubscribe` method of `Subject` — and its derived classes — as it has some surprising behaviour.

## Subscriptions

If you look at the signature for [`Observable.prototype.subscribe`](https://github.com/ReactiveX/rxjs/blob/6.5.3/src/internal/Observable.ts#L73-L80), you’ll see that it returns a `Subscription`. And if you’ve used observables, you will be familiar with calling the subscription’s `unsubscribe` method. However, a subscription contains more than just the `unsubscribe` method.

In particular, the `Subscription` class implements the `SubscriptionLike` interface:

```ts
export interface SubscriptionLike extends AnonymousSubscription {
  unsubscribe(): void;
  readonly closed: boolean;
}
```

Where `AnonymousSubscription` is the same interface, but without the read-only `closed` property.

The `closed` property indicates whether or not the subscription has been unsubscribed — either manually or automatically (if the observable completes or errors).

## Subscribers and unsubscription

Interestingly, what’s actually returned from a call to `subscribe` is an instance of the `Subscriber` class — which extends the `Subscription` class.

Like the `subscribe` method, the `Subscriber` class can be passed a partial observer or individual `next`, `error` and `complete` callback functions.

The primary purpose of a `Subscriber` is to ensure the observer methods or callback functions are called only if they are specified and to ensure that they are not called after `unsubscribe` is called or the source observable `completes` or `errors`.

It’s possible to create a `Subscriber` instance and pass it in a `subscribe` call — as `Subscriber` implements the `Observer` interface. The `Subscriber` will track subscriptions that are effected from such `subscribe` calls and `unsubscribe` can be called on either the `Subscriber` or the returned `Subscription`.

It’s also possible to pass the instance in more than one `subscribe` call and calling `unsubscribe` on the `Subscriber` will unsubscribe it from all observables to which it is subscribed and mark it as closed. Here, calling `unsubscribe` will unsubscribe it from both `one` and `two`:

```ts
import { interval, Subscriber } from "rxjs";
import { map } from "rxjs/operators";

const one = interval(1000).pipe(map((value) => `one(${value})`));
const two = interval(2000).pipe(map((value) => `two(${value})`));

const subscriber = new Subscriber<string>((value) => console.log(value));
one.subscribe(subscriber);
two.subscribe(subscriber);
subscriber.unsubscribe();
```

So what does this have to do with subjects? Well, subjects behave differently.

## Subjects

A subject is both an observer and an observable. The `Subject` class extends the `Observable` class and implements the `Observer` interface. It also implements the `SubscriptionLike` interface — so subjects have a read-only `closed` property and an `unsubscribe` method.

Its implementation of `SubscriptionLike` suggests that — as with a `Subscriber` — it ought to be possible to subscribe and unsubscribe a `Subject`, like this:

```ts
import { interval, Subject } from "rxjs";
import { map } from "rxjs/operators";

const one = interval(1000).pipe(map((value) => `one(${value})`));
const two = interval(2000).pipe(map((value) => `two(${value})`));

const subject = new Subject<string>();
subject.subscribe((value) => console.log(value));

one.subscribe(subject);
two.subscribe(subject);
subject.unsubscribe();
```

However, an error will be effected:

```text
ObjectUnsubscribedError: object unsubscribed
  at new ObjectUnsubscribedError
  at Subject.next
  at SubjectSubscriber.Subscriber.\_next
  at SubjectSubscriber.Subscriber.next
  at MapSubscriber.\_next
  at MapSubscriber.Subscriber.next
  at AsyncAction.IntervalObservable.dispatch
  at AsyncAction.\_execute
  at AsyncAction.execute
  at AsyncScheduler.flush
```

Why? Well, the [`unsubscribe`](https://github.com/ReactiveX/rxjs/blob/5.5.6/src/Subject.ts#L96-L100) method in the `Subject` class doesn’t actually unsubscribe anything. Instead, it marks the subject as `closed` and sets its internal array subscribed observers — `Subject` extends `Observable`, remember — to `null`.

Subjects track the observers that are subscribed to the subject, but unlike subscribers, they do not track the observables to which the subject itself is subscribed — so subjects are unable to unsubscribe themselves from their sources.

So why does the error occur? The error is thrown by the subject when its [`next`](https://github.com/ReactiveX/rxjs/blob/5.5.6/src/Subject.ts#L53-L55), [`error`](https://github.com/ReactiveX/rxjs/blob/5.5.6/src/Subject.ts#L67-L69) or [`complete`](https://github.com/ReactiveX/rxjs/blob/5.5.6/src/Subject.ts#L83-L85) method is called once it has been marked as `closed` and [the behaviour is by design](https://medium.com/@benlesh/on-the-subject-of-subjects-in-rxjs-2b08b7198b93):

> If you want the subject to loudly and angrily error when you next to it after it’s done being useful, you can call unsubscribe directly on the subject instance itself. — Ben Lesh

The behaviour means that if you call `unsubscribe` on a subject, you have to be sure that it has either been unsubscribed from its sources or that the sources have completed or errored.

## Precautions

Given that the behaviour is so surprising, you might want to disallow — or be warned of — calls to `unsubscribe` on subjects. If you do, my [`rxjs-tslint-rules`](https://github.com/cartant/rxjs-tslint-rules) package includes a rule that does just that: `rxjs-no-subject-unsubscribe`.

The rule also prevents subjects from being passed to a subscription’s `add` method — a method that will be the subject of a future article on subscription composition.
