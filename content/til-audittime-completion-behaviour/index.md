---
title: "TIL: auditTime’s Completion Behaviour Is Unlike debounceTime’s"
description: Similar, but different. But similar.
date: "2020-04-28T14:29:00+1000"
categories: []
keywords: []
cardImage: "./title-card.jpeg"
---

![End](title.jpeg "Photo by Markus Spiske on Unsplash")

The `debounceTime` and `auditTime` operators are similar, but different. Each operator ignores intermediate `next` notifications for a specified duration — that defines a time window — and then emits the most-recently received value when that duration elapses.

The fundamental difference is that `debounceTime` resets the its internal timer whenever a `next` notification that is received within the time window and `auditTime` time does not.

These marble diagrams show the difference:

![debounceTime without completion](debouncetime-incomplete-widened.png)

![auditTime without completion](audittime-incomplete-widened.png)

With `debounceTime`, the timer is started when `a` is received and is reset when `b` is received, so the emission of `b` is delayed by the specified number of milliseconds.

With `auditTime`, the timer is started when `a` is received and is not reset when `b` is received, so the emission of `b` occurs earlier than with `debounceTime` — as the delay is relative the the receipt of `a`.

That's fine. I knew that — that's what's in the docs.

What I didn't know was that the completion behaviour differs.

With `debounceTime`, if completion occurs within the time window, the operator completes immediately and emits the most-recently received value. Like this:

![debounceTime with completion](debouncetime-complete-widened.png)

With `auditTime`, if completion occurs within the time window, the operator completes immediately, without emitting the most-recently received value. Like this:

![auditTime with completion](audittime-complete-widened.png)

This makes sense, as `auditTime` only emits when it receives a signal from its notifier — the timer — and its source completes before it receives a signal. However, it wasn't what I had expected.
