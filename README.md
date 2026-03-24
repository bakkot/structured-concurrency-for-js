# Adding structured concurrency to JavaScript

This repository exists for me and anyone else interested to explore directions for adding [structured concurrency](https://sustrik.github.io/250bpm/blog:71/) to JavaScript.

## What is structured concurrency?

"Structured concurrency" refers to a way of writing concurrent programs such that:

- child tasks are bound to a lexical scope
- scope doesn't exit until children complete or shut down
  - including waiting for cleanup to finish if canceled
- (usually) errors in one child task cancel other tasks

## What does it look like in other languages?

### Java

Java has [`StructuredTaskScope`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/StructuredTaskScope.html), designed for use with its [try-with-resources](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html) statement (which is its way of doing explicit resource management, like JS's `using` statement). It looks like this:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
  var userTask   = scope.fork(() -> fetchUser(userId));
  var ordersTask = scope.fork(() -> fetchOrders(userId));
  var recsTask   = scope.fork(() -> fetchRecommendations(userId));

  scope.join();
  scope.throwIfFailed();

  // if we get here, all subtasks have completed
  return "user=%s, orders=%s, recs=%s".formatted(
      userTask.get(), ordersTask.get(), recsTask.get()
  );
} catch (Exception e) {
  // if we get here, all subtasks have been fully shut down
  return "error: failed to load";
}
```

These tasks run in lightweight threads, and rather than having explicit `async`/`await` points there are various blocking library methods which check whether their thread has been interrupted, throwing `InterruptedException` if so.


### Python

Python has [`asyncio.TaskGroup`](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup), designed for use with its `async with` statement (which is its way of doing explicit resource management). It looks like this:

```py
try:
  async with asyncio.TaskGroup() as tg:
    user_task   = tg.create_task(fetch_user(userId))
    orders_task = tg.create_task(fetch_orders(userId))
    recs_task   = tg.create_task(fetch_recommendations(userId))

  # if we get here, all subtasks have completed
  return (f"user={user_task.result()}, "
          f"orders={orders_task.result()}, "
          f"recs={recs_task.result()}")
except Exception:
  # if we get here, all subtasks have been fully shut down
  return "error: failed to load"
```

`create_task` returns a `Task` object which has a `.cancel()` method, which causes the current `await` point to throw a `CancelledError`.


### Swift

Swift's `async`/`await` was designed for structured concurrency from the start; when the scope exits (whether normally or due to an error), any un-awaited `async let` bindings are automatically canceled and then awaited. It looks like this:

```swift
do {
  async let user   = fetchUser(userId)
  async let orders = fetchOrders(userId)
  async let recs   = fetchRecommendations(userId)

  let result = try await "user=\(user), orders=\(orders), recs=\(recs)"
  // if we get here, all subtasks have completed
  return result
} catch {
  // if we get here, all subtasks have been fully shut down
  return "error: failed to load"
}
```

In Swift there is an ambient `Task`, and async functions must explicitly check `Task.isCancelled` at all points at which they want to be cancelable (although many standard library functions do this internally).


## What can we do in JavaScript?

There's prior art here: for example, the [Effection](https://frontside.com/effection) library achieves structured concurrency with no new JS features by asking you to write your async code with _generators_ instead of async functions. This lets its executor interrupt execution at any `await` point (or rather `yield`, since these are generators), which it uses to do cancelation (even triggering `finally` blocks because of how generators work). This gives you structured concurrency much like Python's.

Unfortunately JavaScript[^1] already has a cancelation primitive, [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController), and browsers have expressed disinterest in adding a new one. Also, we are unlikely to convince users to change how they write every function.

[^1]: Pedantically this is not in JavaScript, but it is in the [WinterTC](https://min-common-api.proposal.wintertc.org/#api-interfaces) minimum common API, which is supported across a wide variety of JavaScript runtimes.

As such, my opinion is that the best route forward is to enhance `AbortController` such that it is usable for structured concurrency. That's what this repo explores. However, please do suggest alternatives in the issues!

## Where we are today

Here's how you write concurrent code with `AbortController`:

```js
const controller = new AbortController();
const { signal } = controller;
try {
  const userP   = fetchUser(userId, { signal });
  const ordersP = fetchOrders(userId, { signal });
  const recsP   = fetchRecommendations(userId, { signal });

  const [ user, orders, recs ] = await Promise.all([ userP, ordersP, recsP ]);
  return `user=${user}, orders=${orders}, recs=${recs}`;
} catch (e) {
  return 'error: failed to load';
} finally {
  controller.abort();
}
```
```js
async function fetchUser(userId, { signal }) {
  signal.addEventListener('abort', () => {
    cleanup();
  });

  // or:
  try {
    // ...
    signal.throwIfAborted();
  } finally {
    cleanup();
  }
}
```

This has several downsides, including

1. `signal` must be manually threaded into every subtask, and their subtasks
1. The [guarantees that platform code gets from signals](https://matrixlogs.bakkot.com/TC39_Delegates/2025-11-19#L229) are not available to code you write yourself ([whatwg/dom#1425](https://github.com/whatwg/dom/pull/1425))
1. The catch block runs before other tasks are canceled
1. No one actually writes that `finally`
1. This does not wait for tasks' async cleanup to complete (for this reason it is _not_ structured concurrency)

I'm not proposing to do anything about the first one, but the rest can be addressed by extending `AbortController` and `AbortSignal`, with no new syntax.

## `addAbortCallback`

There is [an open PR](https://github.com/whatwg/dom/pull/1425) to add a `.addAbortCallback(cb)` method to signals, which would allow writing `signal.addAbortCallback(cleanup)` with the same guarantees platform code gets:

- your callback can only be triggered by someone calling `controller.abort`, not by other code mucking about with the signal
- another callback calling `stopImmediatePropagation` won't prevent your callback from being triggered
- the callback is released once the signal is aborted, as opposed to staying un-garbage-collectable until the signal itself is collected
- adding a callback after the signal is already aborted won't do anything (and in particular won't hold a reference to your callback)

This would solve point 2 above.

In the linked thread there is some discussion of whether this should return an "unregister" capability, which would let you remove the callback. I feel strongly that it should, and furthermore the token should have a [`Symbol.dispose`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/dispose) method which removes the callback. Right now [failure to clean up the callback](https://github.com/jasnell/discussion-abort-protocol/issues/7) is a common source of leaks with `AbortSignal`. This would let you write

```js
async function doTasks({ signal }) {
  using _ = signal.addAbortCallback(() => {
    console.log('aborted');
  });

  await task1({ signal });
  await task2({ signal });

  // cleanup handler is automatically removed when we get here, or if either task throws
}
```

## Disposable `AbortController`

Java and Python make use of their explicit resource management mechanisms to tie tasks to a scope. What if we did the same? If `AbortController` had a `Symbol.dispose` method which was an alias for its `.abort()`, we could write this:

```js
try {
  using disposer = new AbortController.Disposable();
  const { signal } = disposer;

  const userP   = fetchUser(userId, { signal });
  const ordersP = fetchOrders(userId, { signal });
  const recsP   = fetchRecommendations(userId, { signal });

  const [ user, orders, recs ] = await Promise.all([ userP, ordersP, recsP ]);
  return `user=${user}, orders=${orders}, recs=${recs}`;
} catch (e) {
  return 'error: failed to load';
}
```

(You [can do this with `DisposableStack`](https://github.com/whatwg/html/issues/8557#issuecomment-3129929601), but it's a lot of boilerplate.)

This would solve point 3 and (hopefully) 4 above. It's still not structured concurrency because of point 5, but it's getting there.

I've [been advocating for this](https://github.com/whatwg/html/issues/8557#issuecomment-1331448189) for a while. I think it's indepenently useful, even without the rest of the stuff here. Note that above I'm suggesting a subclass of `AbortController`, in deference to [my colleague](https://github.com/whatwg/html/issues/8557#issuecomment-1724193449), but personally I'd be fine with making the base `AbortController` work like this.

## Async cleanup

Independently of the above, people [have asked](https://github.com/whatwg/dom/issues/1389) for the ability to wait for async cleanup tasks to complete. We could imagine a version of `AbortController` where `.abort()` returns a Promise which settles only when all promises returned by the cleanup callbacks have settled:

```js
const controller = new AsyncAbortController();
const { signal } = controller;
try {
  const userP   = fetchUser(userId, { signal });
  const ordersP = fetchOrders(userId, { signal });
  const recsP   = fetchRecommendations(userId, { signal });

  const [ user, orders, recs ] = await Promise.all([ userP, ordersP, recsP ]);
  return `user=${user}, orders=${orders}, recs=${recs}`;
} catch (e) {
  return 'error: failed to load';
} finally {
  // tasks add Promise-returning cleanup callbacks to the signal
  // the "await" blocks until those have all finished
  await controller.abort();
}
```

We could even add a `Symbol.asyncDispose` method to this class, letting it be used with `await using`:

```js
try {
  await using controller = new AbortController.AsyncDisposable();
  const { signal } = controller;

  const userP   = fetchUser(userId, { signal });
  const ordersP = fetchOrders(userId, { signal });
  const recsP   = fetchRecommendations(userId, { signal });

  const [ user, orders, recs ] = await Promise.all([ userP, ordersP, recsP ]);
  return `user=${user}, orders=${orders}, recs=${recs}`;
} catch (e) {
  return 'error: failed to load';
}
```

This solves point 5 above, assuming tasks do all of their cleanup inside of the handler (for which, see below). This would give us structured concurrency at least from the point of view of the caller.

## Using the signal

Let's imagine we get an async-disposable `AbortController` as in the previous section. How do you actually use it? Remember, we only have `addEventListener` (or, hopefully, `addAbortCallback`) and `throwIfAborted`. `throwIfAborted` is nice in that it triggers `finally` blocks and `using` cleanup methods, but we need to do our cleanup inside of a callback in order for the async `abort` to work, and that callback cannot run `finally` blocks or `using` cleanup methods. We can work around that in a kind of kludgy way:

```js
async function fetchUser(userId, { signal }) {
  const { promise, resolve } = Promise.withResolvers();
  using _ = signal.addAbortCallback(() => promise);
  try {
    await using db = await connectToDatabase();
    signal.throwIfAborted();
    const page = await db.fetchForUser(userId);
    signal.throwIfAborted();
    return await parse(page);
  } finally {
    resolve();
  }
}
```
but it would be better with a helper:

```js
AbortSignal.prototype.mustComplete = function() {
  const { promise, resolve } = Promise.withResolvers();
  const token = this.addAbortCallback(() => promise);
  return {
    [Symbol.dispose]: {
      token.unregister();
      resolve();
    },
  };
};

async function fetchUser(userId, { signal }) {
  using _ = signal.mustComplete();
  await using db = await connectToDatabase();
  signal.throwIfAborted();
  const page = await db.fetchForUser(userId);
  signal.throwIfAborted();
  return await parse(page);
}
```

Then we have only a single line of boilerplate in the function.


## Alternative: `controller.wrap(promise)`

Instead of requiring functions to register themselves with `signal.mustComplete()`, we could have a `controller.wrap(promise)` which acts as the identity except that it adds the passed Promise to the list of values which must settle before the Promise returned from `.abort()` settles:

```js
try {
  await using controller = new AbortController.AsyncDisposable();
  const { signal } = controller;

  const userP   = controller.wrap(fetchUser(userId, { signal }));
  const ordersP = controller.wrap(fetchOrders(userId, { signal }));
  const recsP   = controller.wrap(fetchRecommendations(userId, { signal }));

  const [ user, orders, recs ] = await Promise.all([ userP, ordersP, recsP ]);
  return `user=${user}, orders=${orders}, recs=${recs}`;
} catch (e) {
  return 'error: failed to load';
}
```

This adds even more boilerplate at the caller but does avoid any additional boilerplate in callees.

## Alternative: `controller.all(promises)`

I no longer consider this idea good and so have hidden it. I got feedback (which I agree with) that people already have trouble internalizing the usage of the Promise combinators, and adding slight variations of all of them would make that situation worse.

<details>
<summary>click to show</summary>

The design above leaves the controller responsible only for cancellation, with task coalescing still done with the usual Promise combinators like `Promise.all`. That works, and I think is my preferred route; it's the simplest design. But it does require the `signal.mustComplete()` boilerplate in callees, which is unfortunate. Another option would be to introduce an `AbortController` version of `Promise.all` which, instead of returning eagerly at the first exception, would instead perform cancellation and continue to wait for the outstanding Promises, and only then throw that exception. Like this:

```js
AbortController.prototype.all = function (promises) {
  const controller = this;
  let firstError;
  let hasError = false;

  const wrapped = Array.from(promises, (p) =>
    Promise.resolve(p).then(
      (value) => value,
      (reason) => {
        if (!hasError) {
          hasError = true;
          firstError = reason;
          controller.abort();
        }
      }
    )
  );

  return Promise.all(wrapped).then((results) => {
    if (hasError) throw firstError;
    return results;
  });
};
```

(Similarly for the other Promise combinators.)

Then you could do

```js
try {
  await using controller = new AbortController.AsyncDisposable();
  const { signal } = controller;

  const userP   = fetchUser(userId, { signal });
  const ordersP = fetchOrders(userId, { signal });
  const recsP   = fetchRecommendations(userId, { signal });

  const [ user, orders, recs ] = await controller.all([ userP, ordersP, recsP ]);
  return `user=${user}, orders=${orders}, recs=${recs}`;
} catch (e) {
  return 'error: failed to load';
}
```

without needing the `mustComplete` helper (since the helper would automatically wait for all the original Promises to settle).

This is inadequate if there's other possible early exits from the block, since the controller does not know about the tasks until they're passed to `controller.all`.

This does have the advantage of not strictly requiring the ability to wait for async `abort` callbacks, if it proves to be infeasible to add that.
</details>

## Downsides

What haven't we accomplished?

The most obvious thing is that the signal needs to be threaded through everything. It would be nice to avoid this, but any such mechanism would be a much bigger change to the language, so I'm setting that aside.

Also, we are in the awkward state where `.throwIfAborted()` triggers `catch` handlers, which generally must re-throw the exception. In ye olden days there was much discussion of having a mechanism to bypass `catch` handlers and trigger `finally` directly, much like the `.return()` method on generators does (which Effection uses), but we were unable to come to a consensus there and I don't think re-visiting that would be productive. So we will need to tell people to call `signal.throwIfAborted()` at the top of their `catch` handlers. Swift survives this mostly by having pattern-matched catch blocks, which [we could conceivably someday get](https://github.com/tc39/proposal-pattern-matching).

Finally, the fact that cancelation uses what you might call an "in-band" signalling mechanism means that there's no obvious way for libraries (including the standard library) to handle generic cancelation, even for tasks they initiate, unless they constrain the signatures of functions they're calling. For example, a [concurrent `AsyncIterator.prototype.map`](https://github.com/tc39/proposal-async-iterator-helpers?tab=readme-ov-file#concurrency) is invoking a generic async mapping function; it would be nice if closing the iterator early (with `.return`) could cancel the tasks created by invoking said function, but there is no obvious way to do this unless the iterator internally creates an `AbortController` and passes its `signal` to the mapping functions.

## Talk to me

This is still very much in the exploratory stage. Most of the things above are independently motivated, so I'm hoping they can be advanced individually rather than needing to be part of a cohesive whole, but I think it would be good to have a cohesive whole to work towards.

Much of this solidifed for me during [a discussion](https://github.com/jasnell/discussion-abort-protocol/issues/7) with @rixtox, for which they have my thanks.

Please open issues with suggestions!
