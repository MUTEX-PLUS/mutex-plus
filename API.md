# mutex-plus API Reference

Synchronization primitives for asynchronous JavaScript and TypeScript.

**Docs:** [https://mutex-plus.github.io/mutex-plus](https://mutex-plus.github.io/mutex-plus)
**Repository:** [https://github.com/MUTEX-PLUS/mutex-plus](https://github.com/MUTEX-PLUS/mutex-plus)
**Author:** [Doug Perez](https://github.com/dougperez69) (`dougperez69@proton.me`)

## Installation

```bash
npm install mutex-plus
```

## Importing

```javascript
// CommonJS
const { Mutex, Semaphore, withTimeout, tryAcquire } = require('mutex-plus');

// ESM
import { Mutex, Semaphore, withTimeout, tryAcquire } from 'mutex-plus';
```

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

## Mutex

Exclusive lock for asynchronous critical sections. Only one holder at a time.

### Constructor

```typescript
const mutex = new Mutex(cancelError?: Error);
```

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `cancelError` | `Error` | `E_CANCELED` | Error used to reject waiters when `cancel()` is called |

### `acquire(priority?: number): Promise<MutexInterface.Releaser>`

Waits until the mutex is free, then takes it. Resolves with an idempotent `release` function that **must** be called when the critical section is finished.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `priority` | `number` | `0` | Higher values are scheduled first |

```typescript
const release = await mutex.acquire();
try {
    // critical section
} finally {
    release();
}
```

### `runExclusive<T>(callback: () => Promise<T> \| T, priority?: number): Promise<T>`

Acquires the mutex, runs `callback`, and releases when the callback settles (resolve, reject, or throw). Returns a promise that adopts the callback result.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `callback` | `() => Promise<T> \| T` | — | Work to run while holding the mutex |
| `priority` | `number` | `0` | Higher values are scheduled first |

```typescript
const result = await mutex.runExclusive(async () => {
    // critical section
    return someValue;
});
```

### `release(): void`

Releases the mutex if it is currently locked. Prefer the releaser from `acquire`, or `runExclusive`.

```typescript
mutex.release();
```

### `waitForUnlock(priority?: number): Promise<void>`

Resolves when a waiter with the given priority could acquire the mutex. Does **not** take the lock.

```typescript
await mutex.waitForUnlock();
```

### `isLocked(): boolean`

`true` while a holder currently owns the mutex.

```typescript
mutex.isLocked();
```

### `cancel(): void`

Rejects every pending waiter with the constructor error (`E_CANCELED` by default). The current holder is not released.

```typescript
mutex.cancel();
```

## Semaphore

Counting semaphore. Up to `initialValue` callers may hold it at once (with default weight `1`).

### Constructor

```typescript
const semaphore = new Semaphore(initialValue: number, cancelError?: Error);
```

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `initialValue` | `number` | — | Starting permit count |
| `cancelError` | `Error` | `E_CANCELED` | Error used to reject waiters when `cancel()` is called |

### `acquire(weight?: number, priority?: number): Promise<[number, SemaphoreInterface.Releaser]>`

Waits until `getValue() >= weight`, then decrements by `weight`. Resolves to `[valueBeforeDecrement, release]`.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `weight` | `number` | `1` | Permits to take. Must be `> 0` |
| `priority` | `number` | `0` | Higher values are scheduled first |

```typescript
const [value, release] = await semaphore.acquire();
try {
    // critical section; `value` is the count before this acquire
} finally {
    release();
}
```

The returned `release` restores the same `weight` automatically and is idempotent.

### `runExclusive<T>(callback: (value: number) => Promise<T> \| T, weight?: number, priority?: number): Promise<T>`

Acquires with the given weight, runs `callback(value)`, and releases when the callback settles.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `callback` | `(value: number) => Promise<T> \| T` | — | Receives the value observed at acquire |
| `weight` | `number` | `1` | Permits to take. Must be `> 0` |
| `priority` | `number` | `0` | Higher values are scheduled first |

```typescript
const result = await semaphore.runExclusive(async (value) => {
    return someValue;
});
```

### `release(weight?: number): void`

Increments the semaphore by `weight` and dispatches queued waiters that can now run.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `weight` | `number` | `1` | Permits to return. Must be `> 0` |

```typescript
semaphore.release();
semaphore.release(3);
```

If you acquired with a weight other than `1`, unscoped `release` must use that same weight. The releaser from `acquire` already does this.

### `waitForUnlock(weight?: number, priority?: number): Promise<void>`

Resolves when a waiter with the given weight and priority could acquire. Does **not** take permits.

```typescript
await semaphore.waitForUnlock(2);
```

### `isLocked(): boolean`

`true` when the current value is `<= 0`.

```typescript
semaphore.isLocked();
```

### `getValue(): number`

Current permit count.

```typescript
const value = semaphore.getValue();
```

### `setValue(value: number): void`

Replaces the permit count, then dispatches waiters that can run under the new value.

```typescript
semaphore.setValue(5);
```

### `cancel(): void`

Rejects every pending waiter with the constructor error. Current holders keep their permits.

```typescript
semaphore.cancel();
```

## Utility functions

### `withTimeout`

Caps how long `acquire`, `runExclusive`, and `waitForUnlock` will wait. The wrapped object keeps the same API as the original mutex or semaphore.

```typescript
function withTimeout(mutex: MutexInterface, timeout: number, timeoutError?: Error): MutexInterface;
function withTimeout(semaphore: SemaphoreInterface, timeout: number, timeoutError?: Error): SemaphoreInterface;
```

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `mutex` / `semaphore` | mutex or semaphore | — | Primitive to decorate |
| `timeout` | `number` | — | Wait limit in milliseconds |
| `timeoutError` | `Error` | `E_TIMEOUT` | Rejection value after the deadline |

```typescript
const mutexWithTimeout = withTimeout(new Mutex(), 100);
const semaphoreWithTimeout = withTimeout(new Semaphore(5), 100, new Error('timed out'));
```

If the lock arrives after the timeout has already rejected, it is released immediately so it cannot leak. A timed-out `runExclusive` does not invoke the callback.

Timeouts apply only to **waiting**. Work that already holds the lock is not interrupted.

### `tryAcquire`

Non-blocking variant: if the lock is not available immediately, reject. Implemented as `withTimeout(..., 0, error)`.

```typescript
function tryAcquire(mutex: MutexInterface, alreadyAcquiredError?: Error): MutexInterface;
function tryAcquire(semaphore: SemaphoreInterface, alreadyAcquiredError?: Error): SemaphoreInterface;
```

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `mutex` / `semaphore` | mutex or semaphore | — | Primitive to decorate |
| `alreadyAcquiredError` | `Error` | `E_ALREADY_LOCKED` | Rejection value when the lock is busy |

```typescript
const nonBlockingMutex = tryAcquire(mutex);
const nonBlockingSemaphore = tryAcquire(semaphore, new Error('busy'));
```

## Errors

Shared singleton `Error` objects. Compare with `===`.

| Export | Default message | Thrown when |
| --- | --- | --- |
| `E_CANCELED` | `request for lock canceled` | `cancel()` rejects a pending waiter |
| `E_TIMEOUT` | `timeout while waiting for mutex to become available` | `withTimeout` deadline elapsed |
| `E_ALREADY_LOCKED` | `mutex already locked` | `tryAcquire` could not take the lock immediately |

Passing your own `Error` into the constructor / decorator replaces the singleton for that instance.

`acquire` / `runExclusive` / `release` / `waitForUnlock` throw a plain `Error` immediately if `weight <= 0`.

## Types

```typescript
interface MutexInterface {
    acquire(priority?: number): Promise<MutexInterface.Releaser>;
    runExclusive<T>(callback: MutexInterface.Worker<T>, priority?: number): Promise<T>;
    waitForUnlock(priority?: number): Promise<void>;
    isLocked(): boolean;
    release(): void;
    cancel(): void;
}

interface SemaphoreInterface {
    acquire(weight?: number, priority?: number): Promise<[number, SemaphoreInterface.Releaser]>;
    runExclusive<T>(callback: SemaphoreInterface.Worker<T>, weight?: number, priority?: number): Promise<T>;
    waitForUnlock(weight?: number, priority?: number): Promise<void>;
    isLocked(): boolean;
    getValue(): number;
    setValue(value: number): void;
    release(weight?: number): void;
    cancel(): void;
}
```
