# AsyncLocalStorage - Web-interoperable Runtimes portable subset

This document describes a portable subset of the Node.js `AsyncLocalStorage` API
that can be implemented by non-Node.js runtimes without dependency on the
lower-level `async_hooks` API.

## Logical model

This subset uses a fundamentally different underlying logical model than what
is *currently* implemented in Node.js' `AsyncLocalStorage` implementation based
on `async_hooks`. I won't go into the details of Node.js' current model here,
however. Instead, I'll focus on the logical model assumed by this subset.

At any given time, there is a current `Async Context Frame` (which, for ease
of referencing I'll just call `frame` from here on out).

The `frame` is effectively an *immutable* map of `storage keys` to `store`
value. We refer to this as the `frame`'s `storage context`. A `storage key`
is associated with exactly one `AsyncLocalStorage` object instance.

There is always a `root frame` whose `storage context` is always empty.

Whenever a new value is associated with a given `storage key`, a new
`frame` is created with the `value` associated with that `storage key`
set in the new frame's storage context.

Whenever a new `frame` is created, it inherits a *copy* of the current
`frame`'s `storage context` before setting the new `value` associated
with the storage key.

In **pseudo-code**, this looks something like:  (this is **NOT** the API,
this is a pesudo-code illustration of the abstract model)

```js
class AsyncLocalStorage {
  #storageKey = {}; // the storage key is opaque and unique to this
                    // ALS instance.
  run(store, fn) {
    return AsyncContextFrame.run(#storageKey, store, fn);
  }

  getStore() {
    return AsyncContextFrame.current().get(this.#storageKey);
  }
}

class AsyncContextFrame {
  static #currentFrame;
  #storageContext = new Map();

  static run(key, store, fn) {
    const current = AsyncContextFrame.current();
    const frame = new AsyncContextFrame(current, key, store);
    return frame.run(fn);
  }

  static current() { return AsyncContextFrame.#currentFrame; }
  static exchange(frame) {
    const prior = AsyncContextFrame.#currentFrame;
    AsyncContextFrame.#currentFrame = frame;
    return prior;
  }

  constructor(current, key, store) {
    this.#storageContext = new Map(current.#storageContext.entries());
    this.#storageContext.set(key, store);
  }

  run(fn) {
    // enter this frame...
    const prior = AsyncContextFrame.exchange(this);
    try {
      return fn();
    } finally {
      AsyncContextFrame:exchange(prior);
    }
  }

  get(key) {
    return this.#storageContext.get(key);
  }
}
```

Any async resource that is created needs only to grab the current `frame`
and run code within its scope to access the correct `storage context`.

Note that key to this model is that the `storage context` is *immutable*
once created. Mutations to the context using `asyncLocalStorage.run()`
use copy-on-write semantics, causing a new `frame` to be created and
entered. This is a key performance optimization over the current model
implemented by Node.js.

The common theme is that the async context should be captured at the moment
an asynchronous task is *scheduled*.

  * `promise.then(fn)` *schedules* a continuation task
  * `setTimeout(fn, n)` *schedules* a timer task
  * `queueMicrotask(fn)` *schedules* a microtask
  * and so on

This is the general rule that runtimes and applications should follow for
any api that schedules asynchronous tasks to run.

## The API

### Class: AsyncLocalStorage

The following is the limited subset of the `AsyncLocalStorage` API:

```js
class AsyncLocalStorage {
  constructor();

  // Creates a new frame, propagating the storage context of
  // the current frame to the new frame, then setting the store
  // as the value associated with this ALS instance. The function
  // is then run within the context of the newly created frame.
  // The return value of calling fn() is returned by run().
  run(store, fn, ...args) : any;

  // In this subset, exit() is equivalent to calling `run(undefined, fn)`
  exit(fn, ...args) : any;

  // Returns the value of store associated with this ALS instance
  // in the current frame's storage context.
  getStore() : any;
}
```

### Class: AsyncResource

The following is the limited subset of the `AsyncResource` API:

```js
class AsyncResource {
  // The type and options here are defined by Node.js, with type
  // *currently* being required by Node.js. In this subset, the
  // type is still required by is *ignored*. It is required so
  // that code written to this subset can be portable to Node.js.
  // The InitOptions current is used to specify async_hooks
  // specific metadata that is not used by this subset. It may
  // be passed but should be ignored by implementations of this
  // subset.
  // On construction, the AsyncResource captures a reference to
  // the current async context frame.
  constructor(type: string, options? : InitOptions);

  // Synchronously calls fn in the scope of the frame captured
  // when the AsyncResource instance was created.
  runInAsyncScope(fn, thisArg?: any, ...args) : any;

  // Returns a new function that wraps the given fn such that when
  // the returned function is called, fn is called in the scope of
  // the frame captured when the AsyncResource instance was created.
  bind(fn, thisArg?: any, ...args) : Function;

  // Creates a new AsyncResource and returns the `resource.bind(fn)`
  // The type argument here is from the Node.js implementation and
  // is passed as the `type` argument to the `AsyncResource`
  // constructor. Implementations of this subset must be capable
  // of accepting the argument but should ignore it.
  static bind(fn, type?: string, thisArg?: any) : Function;
}
```

### Importing `AsyncLocalStorage` and `AsyncResource`

The classes should be importable/requirable using, at least the `node:async_hooks`
specifier and may be accessed using the bare `async_hooks` specifier. For
example:

```
import { AsyncLocalStorage, AsyncResource } from 'node:async_hooks'

// or

const { AsyncLocalStorage, AsyncResource } = require('node:async_hooks');
```

Neither the `AsyncLocalStorage` and `AsyncResource` constructors should
be accessible via the global scope (`globalThis`).

## Capturing the async context

In general terms, an "async resource" is any task that is expected
to capture the current async context frame and run within its scope
at some point in the future. There are some async resources built in
to JavaScript (e.g. promises, thenables, and microtasks), some defined
by Web Platform APIs (e.g. timers, background tasks, unhandled rejections),
some defined by the runtime, and others defined by the application. The
common characterist of all async resources is that they are expected to
capture the current async context frame and propagate it at various
specific times.

### Promises

A promise itself is *not* an async resource. A promise continuation task,
however, *is*. In other words, when we use `then()` to attach a continuation
task to a promise, the current async context is captured and associated with
that continuation task.

Note that this is a variance from Node.js' current implementation which treats
the promise *itself* as the async resource, capturing the context at the moment
the promise is created rather than when the continuation task is created.

***This is an area that will likely continue to evolve and is currently being
actively discussed. We *could* end up with a model similar to Node.js' in the
end but there's still a lot of uncertainty here***


#### Thenables

Thenables (promise-like objects that expose a `then()` method, should be handled
in exactly the same way as a promise, by default.

Thenables do, however, have the option of being defined as extending
`AsyncResource`, overriding the default behavior.

*** As with Promises in general, this is an area that is still up for discussion
and consideration.***

### queueMicrotask

`queueMicrotask()` must capture the async context frame that is current when
`queueMicrotask()` is called and run the callback within the context of that
frame.

### Timers

Timers (using `setTimeout()`, `setInterval()`) must capture the async context
frame that is current when the timer is created and must run the callback
function within the context of that frame.

### General guidelines for runtimes and applications

The common theme is that the async context should be captured at the moment
an asynchronous task is *scheduled*.

  * `promise.then(fn)` *schedules* a continuation task
  * `setTimeout(fn, n)` *schedules* a timer task
  * `queueMicrotask(fn)` *schedules* a microtask
  * and so on

This is the general rule that runtimes and applications should follow for
any api that schedules asynchronous tasks to run.

For example, imagine an API that processes a stream of data by calling
a set of configured callbacks. The context that is propagate to each of
the callbacks should be the context that is current when the processing
it *started*.

```js
const als = new AsyncLocalStorage();

const processor = new Processor({
  onStart() {
    console.log(als.getStore()); // 123
  },
  onEnd() {
    console.log(als.getStore()); // 123
  },
});

als.run(123, () => processor.start(data));
```

Here, the call to `processor.start(data)` actually schedules the async
activity, so that is the context that is propagated by default. If,
alternatively, the intent is for the callbacks to run within the
context that is current when the `Processor` instance is created,
`AsyncResource.bind()` should be used, of the `Processor` can extend
`AsyncResource` and use `this.runInAsyncScope()`.

```js
const als = new AsyncLocalStorage();

const processor = new Processor({
  onStart: AsyncResource.bind(() {
    console.log(als.getStore()); // undefined
  }),
  onEnd: AsyncResource.bind(() {
    console.log(als.getStore()); // undefined
  }),
});

als.run(123, () => processor.start(data));
```

`EventTarget` and `EventEmitter` are slightly different cases. Events
are *technically not* asynchronous tasks. Both EventTarget and
EventEmitter dispatch events fully sychronously so they will always
run in the same context frame as the dispatchEvent/emit call.

```js
const als = new AsyncLocalStorage();
const et = new EventTarget();

als.run(123, () => {
  et.addEventListener('foo', () => {
    console.log(als.getStore()); // 321
  });
});

als.run(321, () => {
  et.dispatchEvent(new Event('foo'));
});
```

For force an event listener to use the context frame that is current
when the listener is added, use `AsyncResource.bind()`:

```js
const als = new AsyncLocalStorage();
const et = new EventTarget();

als.run(123, () => {
  et.addEventListener('foo', AsyncResource.bind(() => {
    console.log(als.getStore()); // 123
  }));
});

als.run(321, () => {
  et.dispatchEvent(new Event('foo'));
});
```

Alternatively, a runtime can provide custom implementations of EventEmitter
or EventTarget that capture and propagate async context in other ways
(for instance, Node.js' `EventEmitterAsyncResource` API).

### Unhandled rejections and the deferred promise pattern

The `unhandledrejection` and `rejectionhandled` events should, by default,
propagate the context that was current when the promise was rejected and *not*
when the promise was created.

(Note that these are the only standard `EventTarget` events that are expected
to propagate async context currently. All other events should be invoked in
the same frame that the `dispatchEvent()` method was called in.)

The deferred promise pattern is an approach to using promises that separates
the `resolve()` and `reject()` functions used to settle the promise from the
actual promise itself.

```js
function deferredPromise() {
  let resolve, reject;
  const promise = new Promise((a,b) => {
    resolve = a;
    reject = b;
  });
  return { promise, resolve, reject }
}
```

Care must be taken when using this pattern to properly propagate the async
context to the Web API `unhandledrejection` and `rejectionhandled` events.

Specifically, given the following example, the `unhandledrejection` event
should, by default, get the value `321` from the `AsyncLocalStorage`
instance, rather than `123`:

```js
const als = new AsyncLocalStorage();

addEventListener('unhandledrejection', (event) => {
  console.log(als.getStore()); //321
});

const { promise, reject } = als.run(123, () => deferredPromise());
als.run(321, () => reject('boom'));
```

If someone really wants the `unhandledrejection` event to receive the
context at the time the promise was created, then the `reject()` should be
wrapped using `AsyncResource.bind()`, for instance:

```js
function deferredPromiseCurrentContext() {
  let resolve, reject;
  const promise = new Promise((a,b) => {
    resolve = AsyncResource.bind(a);
    reject = AsyncResource.bind(b);
  });
  return { promise, resolve, reject }
}
```

This would ensure that whenever `reject()` is called, it enters the
context that was current when the promise was created.

Why is this the case? The `reject()` itself is *not* an asynchronous
operation but it directly *schedules* the emission of the unhandled
reject event. It's *that* scheduling of the event that qualifies as
an asynchronous operation boundary through which the current async
context should be scheduled.
