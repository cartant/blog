---
title: "RxJS: Stopped Notifications"
description: "How to reconfigure stopped notification behaviour"
date: "2020-10-23T14:41:00+1000"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title-card.jpeg"
---

![Stoplight](title.jpeg "Photo by Erwan Hesry on Unsplash")

RxJS — and Rx in general — offers a bunch of guarantees that make it possible to compose observable chains. Specifically, it's guaranteed that:

- observers won't receive notifications after an `error` or `complete` notification; and
- observers won't receive notifications after explicitly unsubscribing.

There is more information about this in the [Observable Contract](http://reactivex.io/documentation/contract.html).

The guarantees allow for the declarative composition of observable chains: developers can focus on composing behaviours rather than keeping track of state. Without the guarantees, developers would be checking for errored or completed sources and composition would be more difficult.

A quirk that stems from these guarantees is that it's possible for an error channel to be closed. For example, if an error occurs when an observable is being torn down, it's not possible for that error to be reported to the observer — the observer will have already received a `complete` or `error` notification or will have already been explicitly unsubscribed. In RxJS, observers in this state are referred to (internally) as _stopped_.

Over the last few RxJS versions, the way this situation has been dealt with has changed.

## Version 5

In RxJS version 5, errors that cannot be reported to stopped observers are — for the most part — swallowed, but synchronous errors that occur as part of the subscription process are re-thrown.

That means that, in some situations, errors that might otherwise be swallowed are thrown from `subscribe`. This behaviour was changed in version 6.

## Version 6

In version 6, the throwing of synchronous errors from `subscribe` is deprecated. That means that the only channel available for reporting subscription errors is the observer. So with stopped observers, errors are swallowed.

This behaviour made determining the cause of a couple of bugs in the core library more difficult than it should have been. So, to give some visibility to errors that are effected within `subscribe`, version 6 traverses the chain of observers and if a stopped observer is found, the error is logged to the console — as a warning. If a stopped observer is not found, the error is sent as a notification.

## Version 7

In version 7, the traversal will no longer be performed. In all situations, errors will be sent to observers as notifications. However, if the observer is stopped, the notification will then be sent to a configurable function, allowing the developer to take an appropriate action. If no configuration is provided — the default — the stopped notifications will be swallowed.

An example configuration looks like this:

```ts
import { config } from "rxjs";

config.onStoppedNotification = (notification) => {
  switch (notification.kind) {
    case "E":
      const error: any = new Error("Cannot notify stopped observer");
      error.notification = notification;
      throw error;
    default:
      return;
  }
};
```

Here, any error notification that cannot be sent to an observer — because it is stopped — will be thrown as an unhandled error.

Note that a `next` or `complete` notification that cannot be sent to an observer will also be routed to any configured `onStoppedNotification` handler. Generally, you won't need to be concerned with those, but their being exposed can be helpful — especially, if you are building your own observable source and are investigating weird behaviour.

To sum up, the new `config` option will let developers decide what should be done with errors that cannot be sent to observers and it has also allowed us to simplify the codebase a little. And swallowed errors are probably my number-one pet peeve, so ... ❤
