---
title: "RxJS: v7 TestScheduler Features"
description: "A sneak peek at some new features for scheduled observables"
date: "2020-09-30T18:16:00+1000"
categories: []
keywords: []
ckTags: ["1464979"]
cardImage: "./title-card.jpeg"
---

![Stethoscope](title.jpeg "Photo by Hush Naidoo on Unsplash")

Let's take a peek at some of the features that have been added to the `TestScheduler` for RxJS version 7 beta.

## Schedulers

In version 6, when the `TestScheduler` is used in run mode, each scheduler implementation is delegated to the `TestScheduler` — so each scheduler behaves exactly the same way. That means, when under test, the schedulers don't behave as they would in an application.

Let's take a look at a test to see the problem:

```ts
const testScheduler = new TestScheduler((a, e) => expect(a).toStrictEqual(e));
testScheduler.run(({ cold, expectObservable, time }) => {
  const tb = time(" ------|  ");
  const expected = "(acd)-b--";

  const result = merge(
    of("a").pipe(delay(0, animationFrameScheduler)),
    of("b").pipe(delay(tb, asyncScheduler)),
    of("c").pipe(delay(0, asyncScheduler)),
    of("d").pipe(delay(0, asapScheduler))
  );
  expectObservable(result).toBe(expected);
});
```

Here, we are merging a bunch of characters — `a` through to `d` — each delayed using a different scheduler or a different delay. `b` is scheduled with a non-zero delay — specifying the frame in which it will be emitted — but when the other characters will be emitted — and the order in which those emissions will occur — depends upon the schedulers themselves.

In version 6, the test passes, but look at that expectation: `c` is emitted before `d`. That's not what would happen in an application.

`c` is delayed using the `asyncScheduler` — which schedules a macro task — and `d` is delayed using the `asapScheduler` — which schedules a micro task — so `d` should be emitted _before_ `c`.

And `a` — delayed to coincide with an animation frame — is emitted before either `c` or `d`. That wouldn't happen in an application.

In RxJS version 7, the `TestScheduler` implementation has been changed and — when in run mode — the delegation is performed at the API level, so micro-task-scheduled notifications will be emitted _before_ macro-task-scheduled notifications.

Putting the `animationFrameScheduler` to one side, the passing test would look like this, in version 7:

```ts
const testScheduler = new TestScheduler((a, e) => expect(a).toStrictEqual(e));
testScheduler.run(({ cold, expectObservable, time }) => {
  const tb = time(" ------|  ");
  const expected = "(dc)--b--";

  const result = merge(
    of("b").pipe(delay(tb, asyncScheduler)),
    of("c").pipe(delay(0, asyncScheduler)),
    of("d").pipe(delay(0, asapScheduler))
  );
  expectObservable(result).toBe(expected);
});
```

Note that although the `asapScheduler`-delayed observable is merged _after_ the `asyncScheduler`-delayed observable, its notification is emitted _before_ the `asyncScheduler`-delayed observable's notification.

In version 7, the `TestScheduler` emits each micro-task-scheduled notification within a frame _before_ it emits any of the frame's macro-task-scheduled notifications.

So what about the `animationFrameScheduler`? How will it behave with the `TestScheduler` in version 7?

If we add the `animationFrameScheduler`-delayed observable to the test, we'll get an error:

```text
Error: animate() was not called within run()
```

Hmm. Interesting. What's that `animate` function mentioned in the error message?

It's a new function that has been added to the run helpers and it can be used like this:

```ts
const testScheduler = new TestScheduler((a, e) => expect(a).toStrictEqual(e));
testScheduler.run(({ animate, cold, expectObservable, time }) => {
  animate("         --------x");
  const tb = time(" ------|  ");
  const expected = "(dc)--b-a";

  const result = merge(
    of("a").pipe(delay(0, animationFrameScheduler)),
    of("b").pipe(delay(tb, asyncScheduler)),
    of("c").pipe(delay(0, asyncScheduler)),
    of("d").pipe(delay(0, asapScheduler))
  );
  expectObservable(result).toBe(expected);
});
```

`animate` takes a marble diagram and each notification in the diagram indicates when an animation frame is to occur — i.e. the notifications represent browser repaints. This gives the test author control over the scheduling of animation frames within the test.

Note that in our test, the `animationFrameScheduler`-delayed observable is merged first. However, its notification is not emitted until the `animates` marble diagram emits a notification.

## animationFrames

The `animate` run helper isn't just for schedulers. It can be used to author tests that use version 7's new `animationFrames` observable, too.

The `animationFrames` observable acts as a source of animation frame notifications. Each time the browser repaints, the `animationFrames` observable emits a `next` notification — the value is comprised of the time `elapsed` since subscription and the [`timestamp` that's passed by the browser](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame) to the `requestAnimationFrame` callback.

With previous versions of RxJS, you might have seen animation-frame sources composed like this:

```ts
const af = interval(0, animationFrameScheduler);
```

Or like this:

```ts
const af = of(0, animationFrameScheduler).pipe(repeat());
```

Not only is it less clear what these composed observables are doing, they don't provide the elapsed time or timestamp information that's actually needed for controlling animations.

In a test, we can use the `animates` helper to control when animation frames occur, like this:

```ts
const testScheduler = new TestScheduler((a, e) => expect(a).toStrictEqual(e));
testScheduler.run(({ animate, expectObservable, time }) => {
  animate("            -----x--x--x");
  const subs = "       --^--------!";
  const ts = time("    --|         ");
  const ta = time("    -----|      ");
  const tb = time("    --------|   ");
  const tc = time("    -----------|");
  const expected = "   -----a--b--c";

  const result = animationFrames();
  expectObservable(result, subs).toBe(expected, {
    a: { elapsed: ta - ts, timestamp: ta },
    b: { elapsed: tb - ts, timestamp: tb },
    c: { elapsed: tc - ts, timestamp: tc }
  });
});
```

Here, the subscription occurs at `ts`, so that's what's used as the basis for the `elapsed` times.

We're pretty pleased with the way the `animates` helper and the API-level delegation have turned out. These new features have eliminated a couple of pain points with the `TestScheduler`. And if you have the version-7 beta installed, you can try these out right now.
