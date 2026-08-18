---
title: mutex-plus
nav_order: 1
description: Mutex and semaphore primitives for asynchronous JavaScript and TypeScript
permalink: /
---

# mutex-plus

Mutex and semaphore primitives for asynchronous JavaScript and TypeScript.

JavaScript is single-threaded, but `await` still yields to the event loop. Two async functions can interleave and race on shared state. **mutex-plus** gives you exclusive-access and limited-concurrency locks that fit native `async`/`await`.

[Get started]({{ site.baseurl }}/guide){: .btn .btn-primary }
[API reference]({{ site.baseurl }}/api){: .btn }

## Install

```bash
npm install mutex-plus
```

```typescript
import { Mutex, Semaphore, withTimeout, tryAcquire } from 'mutex-plus';
```

## Exclusive access

```typescript
import { Mutex } from 'mutex-plus';

const mutex = new Mutex();
let balance = 100;

async function withdraw(amount: number) {
    return mutex.runExclusive(async () => {
        await chargeCard(amount);
        balance -= amount;
        return balance;
    });
}
```

## Bounded parallelism

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

## What you get

- **Mutex** — one holder at a time across `await` points
- **Semaphore** — up to *n* concurrent holders, with optional weights
- **`runExclusive`** — acquire, run, always release
- **Priority queue** — higher-priority waiters run first
- **`withTimeout`** / **`tryAcquire`** — deadline or fail-immediately
- **`cancel()`** — reject pending waiters without unlocking the holder
- Native TypeScript types, CommonJS, and ESM

## Links

- [Usage guide]({{ site.baseurl }}/guide)
- [API reference]({{ site.baseurl }}/api)
- [GitHub repository](https://github.com/MUTEX-PLUS/mutex-plus)
- [npm package](https://www.npmjs.com/package/mutex-plus)
- Maintainer: [Doug Perez](https://github.com/dougperez69) (`dougperez69@proton.me`)
