[![Build and Tests](https://github.com/MUTEX-PLUS/mutex-plus/actions/workflows/build_and_test.yaml/badge.svg)](https://github.com/MUTEX-PLUS/mutex-plus/actions/workflows/build_and_test.yaml)
[![npm version](https://img.shields.io/npm/v/mutex-plus.svg)](https://www.npmjs.com/package/mutex-plus)
[![types](https://img.shields.io/npm/types/mutex-plus.svg)](https://www.npmjs.com/package/mutex-plus)
[![license](https://img.shields.io/npm/l/mutex-plus.svg)](LICENSE)

# mutex-plus

Mutex and semaphore primitives for asynchronous JavaScript and TypeScript.

JavaScript is single-threaded, but `await` still yields to the event loop. Two async functions can interleave, race on shared state, and leave your program in a state that is hard to reproduce. **mutex-plus** gives you the same exclusive-access and limited-concurrency tools you would use on real threads — expressed as promises, so they fit native `async`/`await` code.

**Documentation:** [https://mutex-plus.github.io/mutex-plus](https://mutex-plus.github.io/mutex-plus)
**API reference:** [API.md](API.md)

## Table of contents

- [Why this exists](#why-this-exists)
- [Features](#features)
- [Installation](#installation)
- [Importing](#importing)
- [Quick start](#quick-start)
- [Mutex](#mutex)
  - [Recommended: `runExclusive`](#recommended-runexclusive)
  - [Manual `acquire` / `release`](#manual-acquire--release)
  - [Unscoped `release()`](#unscoped-release)
  - [`isLocked()`](#islocked)
  - [`waitForUnlock()`](#waitforunlock)
  - [`cancel()`](#cancel)
  - [Priority](#priority)
- [Semaphore](#semaphore)
  - [Creating](#creating)
  - [Recommended: `runExclusive`](#recommended-runexclusive-1)
  - [Manual `acquire` / `release`](#manual-acquire--release-1)
  - [Weights](#weights)
  - [Unscoped `release(weight)`](#unscoped-releaseweight)
  - [`getValue()` / `setValue()`](#getvalue--setvalue)
  - [`isLocked()`](#islocked-1)
  - [`waitForUnlock(weight, priority)`](#waitforunlockweight-priority)
  - [`cancel()`](#cancel-1)
- [Timeouts](#timeouts)
- [Fail immediately: `tryAcquire`](#fail-immediately-tryacquire)
- [Errors](#errors)
- [TypeScript](#typescript)
- [Runtime support](#runtime-support)
- [Best practices](#best-practices)
- [License](#license)

## Why this exists

A mutex on a multi-threaded system blocks a thread until it owns the lock. JavaScript cannot block a thread that way: while you `await` a network call, a timer, or a worker message, other code on the same event loop can run.

That is enough to create races. A typical lost-update looks like this:

```typescript
let balance = 100;

async function withdraw(amount: number) {
    const current = balance;      // read
    await chargeCard(amount);     // yield — other callers can enter here
    balance = current - amount;   // write — previous update can be overwritten
}
```

If `withdraw(30)` and `withdraw(40)` overlap, both may read `100` and the final balance can be `70` or `60` instead of `30`.

mutex-plus applies mutual exclusion to that async gap. Locking returns a promise that resolves when it is safe to enter the critical section. You do the async work, then release. The next waiter runs only after that.

A **semaphore** is the same idea with a count. Initialize it with `n` and up to `n` callers may hold the lock at once — useful for worker pools, crawlers, and bounded parallelism.

## Features

- **Mutex** — exclusive access to a critical section across `await` points
- **Semaphore** — allow up to *n* concurrent holders, with optional weights
- **`runExclusive`** — acquire, run, release (including on throw / rejection)
- **Priority queue** — higher-priority waiters are scheduled first
- **`withTimeout`** — cap how long you wait for a lock
- **`tryAcquire`** — fail immediately if the lock is not free
- **`cancel()`** — reject every pending waiter without unlocking a holder
- **`waitForUnlock()`** — observe availability without taking the lock
- Native **TypeScript** types, **CommonJS**, **ESM**, and bundler-friendly builds
- Zero runtime dependencies besides `tslib`

## Installation

```bash
npm install mutex-plus
```

```bash
yarn add mutex-plus
```

```bash
pnpm add mutex-plus
```

The package ships its own TypeScript declarations. No `@types` package is required.

## Importing

**CommonJS**

```javascript
const { Mutex, Semaphore, withTimeout, tryAcquire } = require('mutex-plus');
```

**ESM**

```javascript
import { Mutex, Semaphore, withTimeout, tryAcquire } from 'mutex-plus';
```

**TypeScript**

```typescript
import {
    Mutex,
    MutexInterface,
    Semaphore,
    SemaphoreInterface,
    withTimeout,
    tryAcquire,
    E_TIMEOUT,
    E_ALREADY_LOCKED,
    E_CANCELED,
} from 'mutex-plus';
```

## Quick start

Serialize access to a shared resource:

```typescript
import { Mutex } from 'mutex-plus';

const mutex = new Mutex();
let balance = 100;

async function withdraw(amount: number) {
    return mutex.runExclusive(async () => {
        if (balance < amount) {
            throw new Error('insufficient funds');
        }
        await chargeCard(amount);
        balance -= amount;
        return balance;
    });
}
```

Limit how many jobs run at once:

```typescript
import { Semaphore } from 'mutex-plus';

const pool = new Semaphore(4);

async function fetchAll(urls: string[]) {
    return Promise.all(
        urls.map((url) =>
            pool.runExclusive(async () => {
                const response = await fetch(url);
                return response.json();
            })
        )
    );
}
```

## Mutex

A mutex lets **one** caller into the critical section at a time. Everyone else waits in a queue.

```typescript
const mutex = new Mutex();
```

Pass an optional `Error` to use when pending waiters are cancelled (see [`cancel()`](#cancel)):

```typescript
const mutex = new Mutex(new Error('lock cancelled'));
```

### Recommended: `runExclusive`

This is the safest API. mutex-plus acquires the lock, runs your callback, and always releases — success, throw, or rejection.

```typescript
const result = await mutex.runExclusive(async () => {
    await writeFile(path, data);
    return 'ok';
});
```

Promise style:

```javascript
mutex
    .runExclusive(() => writeFile(path, data))
    .then(() => {
        /* released */
    });
```

The callback may return a value or a promise. The promise returned by `runExclusive` adopts that result. If the callback throws or rejects, the mutex is still released and the error propagates.

An optional **priority** argument is accepted (higher number runs first; default `0`):

```typescript
await mutex.runExclusive(async () => {
    /* ... */
}, 10);
```

### Manual `acquire` / `release`

When you need the lock to span a larger scope than a single callback:

```typescript
const release = await mutex.acquire();
try {
    await writeFile(path, data);
} finally {
    release();
}
```

Promise style:

```javascript
mutex.acquire().then((release) => {
    return doWork().finally(() => release());
});
```

`acquire` resolves with an idempotent `release` function. Calling it twice is a no-op.

**You must call `release`.** If you forget, the mutex stays locked and every later waiter deadlocks. Prefer `runExclusive` unless you have a reason not to.

`acquire` also accepts a priority:

```typescript
const release = await mutex.acquire(5);
```

### Unscoped `release()`

You can release the current holder without the callback from `acquire`:

```typescript
mutex.release();
```

This is convenient in some state machines, but it is easy to release the *wrong* lock if more than one code path calls it. Prefer the releaser returned by `acquire`, or `runExclusive`.

### `isLocked()`

```typescript
if (mutex.isLocked()) {
    // someone currently holds the mutex
}
```

This is a snapshot. By the time you act on it, the state may have changed. Do not use it as a substitute for `acquire` / `tryAcquire`.

### `waitForUnlock()`

Wait until a lock *could* be taken, without taking it:

```typescript
await mutex.waitForUnlock();
```

There is no guarantee it is still free after the next `await`. Use this for back-pressure or UI (“queue is moving”), not for exclusive access.

### `cancel()`

Reject every **pending** waiter. The current holder is not forced to release.

```typescript
mutex.cancel();
```

Waiters fail with `E_CANCELED`, or with the error you passed to the constructor:

```typescript
import { E_CANCELED } from 'mutex-plus';

try {
    await mutex.runExclusive(async () => {
        /* ... */
    });
} catch (error) {
    if (error === E_CANCELED) {
        // this waiter was cancelled
    }
}
```

After `cancel()`, the mutex may still be locked by the current holder.

### Priority

Waiters are a priority queue. A larger `priority` value is served before a smaller one. The default is `0`. Negative values are allowed.

```typescript
await mutex.acquire(-1); // low
await mutex.acquire(0);  // default
await mutex.acquire(10); // high — jumps the queue
```

A task that can run immediately (lock is free, queue empty) is not delayed just because a higher-priority task appears later — it already started. Priority applies to **queued** waiters.

## Semaphore

A semaphore is a mutex with a count. Create it with the number of permits you want to hand out at once.

Typical uses:

- at most *n* parallel HTTP requests
- a pool of *n* workers
- weighted admission (a large job takes more than one permit)

### Creating

```typescript
const semaphore = new Semaphore(5);
```

`5` means five callers may hold the semaphore at the same time. A mutex is a semaphore of `1`.

Optional cancel error:

```typescript
const semaphore = new Semaphore(5, new Error('pool cancelled'));
```

The value may become negative if you over-acquire via `setValue` / `release` combinations; `isLocked()` is true when the value is `<= 0`.

### Recommended: `runExclusive`

```typescript
await semaphore.runExclusive(async (value) => {
    // `value` is the semaphore count *before* this acquire decremented it
    return transform(image);
});
```

The callback receives the value observed at acquire time. The semaphore is released when the callback settles.

Optional weight and priority:

```typescript
await semaphore.runExclusive(
    async () => heavyJob(),
    3,  // weight: wait until 3 permits are free
    10  // priority
);
```

### Manual `acquire` / `release`

```typescript
const [value, release] = await semaphore.acquire();
try {
    await transform(image);
} finally {
    release();
}
```

The resolved tuple is `[currentValue, release]`. `currentValue` is the count **before** this acquire subtracted its weight. `release` is idempotent and returns the correct weight automatically.

```typescript
const [value, release] = await semaphore.acquire(2, 5); // weight 2, priority 5
```

### Weights

`weight` is how many permits this waiter needs. It must be a **positive** number. The waiter runs only when `semaphore.getValue() >= weight`.

```typescript
const gpu = new Semaphore(8);

await gpu.runExclusive(() => inferSmall(), 1);
await gpu.runExclusive(() => inferLarge(), 4);
```

Use weights when jobs consume different amounts of a shared budget (memory, connections, GPU slots).

### Unscoped `release(weight)`

```typescript
semaphore.release();    // +1
semaphore.release(3);   // +3
```

The releaser from `acquire` already knows the weight. If you call `semaphore.release()` yourself, **you** must pass the same weight you acquired. Mismatching weights will leak permits or starve the queue.

### `getValue()` / `setValue()`

```typescript
semaphore.getValue();
semaphore.setValue(4);
```

`setValue` replaces the count and then schedules any waiters that can now run. Use it for reconfiguration (for example, shrinking a pool), not as a replacement for `release`.

### `isLocked()`

```typescript
semaphore.isLocked(); // true when value <= 0
```

### `waitForUnlock(weight, priority)`

Resolves when a caller with the given weight and priority *could* acquire, without acquiring:

```typescript
await semaphore.waitForUnlock(2);
```

Same caveat as the mutex: availability can change at the next async boundary.

### `cancel()`

Rejects pending waiters with `E_CANCELED` (or the constructor error). Current holders keep their permits.

```typescript
semaphore.cancel();
```

## Timeouts

Wrap a mutex or semaphore so `acquire`, `runExclusive`, and `waitForUnlock` give up after a deadline.

```typescript
import { withTimeout, E_TIMEOUT } from 'mutex-plus';

const mutex = withTimeout(new Mutex(), 100);
const semaphore = withTimeout(new Semaphore(5), 100);
```

The decorated object has the same methods as the original. After `timeout` milliseconds, the waiting promise rejects with `E_TIMEOUT` and `runExclusive` does **not** run the callback.

If the lock is granted after the timeout has already fired, mutex-plus releases it immediately so the lock cannot leak.

Custom error:

```typescript
const mutex = withTimeout(new Mutex(), 100, new Error('lock timed out'));
```

```typescript
try {
    await mutex.runExclusive(async () => {
        /* ... */
    });
} catch (error) {
    if (error === E_TIMEOUT) {
        // did not obtain the lock in time
    }
}
```

Timeouts apply to **waiting**, not to the work inside the critical section. Once you hold the lock, your callback runs to completion (or rejection) with no extra timer.

## Fail immediately: `tryAcquire`

Do not queue at all. If the lock is not free right now, reject.

```typescript
import { tryAcquire, E_ALREADY_LOCKED } from 'mutex-plus';

try {
    await tryAcquire(mutex).runExclusive(() => {
        // runs only if the mutex was free
    });
} catch (error) {
    if (error === E_ALREADY_LOCKED) {
        // someone else holds it
    }
}
```

Works on both mutexes and semaphores. Custom error:

```typescript
tryAcquire(mutex, new Error('busy')).runExclusive(() => {
    /* ... */
});
```

`tryAcquire` is implemented as a zero-millisecond `withTimeout`. It is the right tool for “skip if busy” paths (optional cache refresh, opportunistic flush), not for work that must eventually run.

## Errors

All three values are shared singleton `Error` objects. Compare with `===`:

| Export | When it is thrown |
| --- | --- |
| `E_CANCELED` | A pending `acquire` / `runExclusive` was cancelled via `cancel()` |
| `E_TIMEOUT` | `withTimeout` deadline elapsed while waiting |
| `E_ALREADY_LOCKED` | `tryAcquire` ran while the lock was unavailable |

```typescript
import { E_CANCELED, E_TIMEOUT, E_ALREADY_LOCKED } from 'mutex-plus';
```

You can replace any of them by passing your own `Error` into `new Mutex`, `new Semaphore`, `withTimeout`, or `tryAcquire`. Identity comparison then uses *your* object.

Invalid semaphore weights (`weight <= 0`) throw a plain `Error` immediately — they never queue.

## TypeScript

mutex-plus is written in TypeScript and publishes `lib/index.d.ts`.

- `MutexInterface` / `SemaphoreInterface` — the surface you should type against if you accept either primitive
- `MutexInterface.Releaser` — `() => void`
- `MutexInterface.Worker<T>` — `() => Promise<T> \| T`
- `SemaphoreInterface.Worker<T>` — `(value: number) => Promise<T> \| T`

`withTimeout` and `tryAcquire` preserve mutex vs semaphore overloads, so a wrapped semaphore still has `getValue` / `setValue` / weighted `acquire`.

## Runtime support

- **Node.js** — CommonJS (`require`) and native ESM (`import`). Native ESM needs Node 12.16+ or 13.7+. Node 13.0–13.1 cannot import this package correctly.
- **Browsers** — ES5 plus `Promise` and `Array.isArray`. Ancient engines need a Promise shim (for example [core-js](https://github.com/zloirock/core-js)).
- **Bundlers / React Native** — `module` and `exports` entry points are provided.

There are no timers inside `Mutex` / `Semaphore` themselves. `withTimeout` / `tryAcquire` use `setTimeout`.

## Best practices

1. **Prefer `runExclusive`.** It is the only API that cannot forget to release.
2. **Always `release` in `finally`** if you use `acquire`.
3. **Do not hold a lock across unrelated work.** Acquire, do the shared-state update, release. Start slow I/O that does not touch the guarded resource *outside* the critical section when you can.
4. **One mutex per resource.** A global mutex serializes unrelated work and kills throughput.
5. **Never lock A then B in one path and B then A in another** if both mutexes can be held at once — that is a deadlock. If you need two locks, always take them in the same order, or collapse them into one.
6. **Do not use `isLocked()` as a lock.** It is informational. Use `tryAcquire` if you need a non-blocking attempt.
7. **Match semaphore weights.** The `acquire` releaser does this for you. Unscoped `release(weight)` does not.
8. **`cancel()` does not preempt the holder.** Drain or time out in-flight work separately if you need that.

## License

mutex-plus is released under the [MIT License](LICENSE).

Maintained by [Doug Perez](https://github.com/dougperez69) (`dougperez69@proton.me`).
