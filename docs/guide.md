---
title: Guide
nav_order: 2
permalink: /guide
---

# Guide

This page covers how to use mutex-plus in application code. For method signatures, see the [API reference]({{ site.baseurl }}/api).

## Why a mutex in single-threaded JavaScript?

While you `await`, other tasks on the same event loop can run. That is enough to lose updates:

```typescript
let balance = 100;

async function withdraw(amount: number) {
    const current = balance;
    await chargeCard(amount);
    balance = current - amount;
}
```

If two `withdraw` calls overlap, both may read `100`. Wrap the read/await/write in a mutex so only one caller is inside that section at a time.

## Mutex

```typescript
import { Mutex } from 'mutex-plus';

const mutex = new Mutex();
```

### `runExclusive` (preferred)

Acquires, runs your callback, and always releases — including on throw or rejection.

```typescript
await mutex.runExclusive(async () => {
    await writeFile(path, data);
});
```

### Manual acquire

Use this when the critical section cannot be a single callback. Always release in `finally`.

```typescript
const release = await mutex.acquire();
try {
    await writeFile(path, data);
} finally {
    release();
}
```

Forgetting `release()` deadlocks every later waiter. The releaser is idempotent: calling it twice is safe.

### Inspect and wait

```typescript
mutex.isLocked();
await mutex.waitForUnlock();
```

`waitForUnlock` does **not** take the lock. After the next `await`, someone else may have acquired it.

### Cancel pending waiters

```typescript
import { E_CANCELED } from 'mutex-plus';

mutex.cancel();
```

Pending `acquire` / `runExclusive` promises reject with `E_CANCELED` (or the `Error` passed to `new Mutex(error)`). The current holder is not forced to unlock.

### Priority

Larger `priority` values run first. Default is `0`.

```typescript
await mutex.acquire(10);
await mutex.runExclusive(async () => {}, 10);
```

## Semaphore

A semaphore is a mutex with a count. Pass how many callers may hold it at once.

```typescript
import { Semaphore } from 'mutex-plus';

const pool = new Semaphore(4);
```

### Parallelism cap

```typescript
await Promise.all(
    jobs.map((job) => pool.runExclusive(() => job()))
);
```

### Weights

A waiter only runs when `getValue() >= weight`. Weight must be positive.

```typescript
const gpu = new Semaphore(8);

await gpu.runExclusive(() => inferSmall(), 1);
await gpu.runExclusive(() => inferLarge(), 4);
```

`acquire` resolves to `[valueBeforeDecrement, release]`. The returned `release` restores the same weight. If you call `semaphore.release(weight)` yourself, you must pass the matching weight.

```typescript
const [value, release] = await semaphore.acquire(2);
try {
    await work();
} finally {
    release();
}
```

### Count

```typescript
semaphore.getValue();
semaphore.setValue(2);
semaphore.isLocked(); // true when value <= 0
```

`setValue` replaces the count and then wakes waiters that can now run.

## Timeouts

```typescript
import { withTimeout, E_TIMEOUT } from 'mutex-plus';

const mutex = withTimeout(new Mutex(), 100);

try {
    await mutex.runExclusive(async () => {
        /* ... */
    });
} catch (error) {
    if (error === E_TIMEOUT) {
        // lock was not obtained in time
    }
}
```

The wrapper keeps the original API. If the lock arrives after the deadline, it is released immediately so it cannot leak. Timeouts apply to **waiting**, not to work already inside the critical section.

## Fail immediately

```typescript
import { tryAcquire, E_ALREADY_LOCKED } from 'mutex-plus';

try {
    await tryAcquire(mutex).runExclusive(() => {
        /* runs only if free */
    });
} catch (error) {
    if (error === E_ALREADY_LOCKED) {
        // already held
    }
}
```

## Errors

Compare with `===` — these are singleton objects:

| Export | Meaning |
| --- | --- |
| `E_CANCELED` | `cancel()` rejected this waiter |
| `E_TIMEOUT` | `withTimeout` deadline elapsed |
| `E_ALREADY_LOCKED` | `tryAcquire` found the lock busy |

You can replace any of them by passing your own `Error` into the constructor or decorator.

## Practices that avoid deadlocks

1. Prefer `runExclusive` so release cannot be skipped.
2. If you use `acquire`, always `release` in `finally`.
3. Keep critical sections short. Do not hold a lock across unrelated I/O.
4. One mutex per resource — a process-wide lock serializes unrelated work.
5. If you must take two mutexes, always take them in the same order.
6. Do not treat `isLocked()` as a lock. Use `tryAcquire` for a non-blocking attempt.
7. `cancel()` does not preempt the current holder.
