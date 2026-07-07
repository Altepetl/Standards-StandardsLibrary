---
title: Rust - Concurrency and Thread Safety
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Concurrency and Thread Safety

> Rust's headline feature is **"fearless concurrency"**: the ownership and borrowing system prevents **data races at compile time** via the `Send` and `Sync` marker traits. This section documents the native model (threads, channels, locks, atomics) and the dominant async ecosystem, presenting recognized alternatives side by side (Tokio vs async-std vs smol; std mpsc vs crossbeam vs flume). It does not pick a runtime winner. Boundary: concurrency *within a process* here; *between* processes (queues, event streams) is in `rust/messaging/` (planned). Canonical source: The Rust Book ch. 16 — https://doc.rust-lang.org/book/ch16-00-concurrency.html, and the Async Book — https://rust-lang.github.io/async-book/.

## 1. The compile-time guarantee: `Send` and `Sync`

| Trait | Meaning |
|---|---|
| `Send` | A type `T` is safe to **move** across thread boundaries (transfer ownership). |
| `Sync` | A type `&T` is safe to **share** between threads (i.e., `&T` is `Send`). |

Most types are auto-`Send`/`Sync` when their contents are. The compiler derives them automatically; you opt **out** with `PhantomData<*const ()>` or `std::cell::Cell`/`RefCell` (which are `!Sync`).

✅ Lean on the compiler — if it compiles, you have no data races:
```rust
use std::thread;
let v = vec![1, 2, 3];
let h = thread::spawn(move || println!("{:?}", v));   // v moved; safe
h.join().unwrap();
```

❌ Trying to share a `Rc<T>` across threads — compile error, because `Rc` is `!Send`/`!Sync`. Use `Arc<T>` instead.

## 2. `std::thread` — OS threads

`std::thread::spawn` creates an OS thread. Each thread owns its stack; `join()` waits for completion. Use `move` closures to transfer ownership into the thread.

✅ Spawn and join:
```rust
let handle = thread::spawn(move || {
    do_work();
});
handle.join().expect("thread panicked");
```

❌ Spawning a thread and never joining or detaching intentionally — if it panics, the panic is silently lost unless you join.

## 3. Message passing — channels

> "Do not communicate by sharing memory; instead, share memory by communicating." (The Rust Book, ch. 16.) Rust culture favors channels for many concurrency designs, but shared-state (`Arc<Mutex<T>>`) is equally idiomatic where it fits.

Recognized channel crates, side by side:

| Crate | Characteristics |
|---|---|
| `std::sync::mpsc` | Multi-producer, single-consumer, in std. Good default for simple cases. The receiver is single-owner. |
| `crossbeam-channel` | Multi-producer multi-consumer, `select!`, very performant. Widely used. |
| `flume` | Fast, ergonomic, async/sync in one crate. Popular newer option. |
| `tokio::sync::mpsc` | The async channel for `tokio`-based runtimes. |

✅ `crossbeam` or `flume` for sync MPMC; `tokio::sync::mpsc` inside an async runtime:
```rust
use std::sync::mpsc;
let (tx, rx) = mpsc::channel();
thread::spawn(move || tx.send(42).unwrap());
println!("{}", rx.recv().unwrap());
```

❌ Sharing the single std `Receiver` across threads — `std::sync::mpsc::Receiver` is `!Sync`. Use `crossbeam`/`flume` for multi-consumer.

## 4. Shared state — `Mutex`, `RwLock`, `Arc`

For shared mutable state, wrap it in `Arc<Mutex<T>>` (or `Arc<RwLock<T>>` for read-heavy access). `parking_lot` is the recognized faster, non-poisoning alternative.

✅ Share mutable state with `Arc<Mutex<T>>`:
```rust
use std::sync::{Arc, Mutex};
let counter = Arc::new(Mutex::new(0));
let handles: Vec<_> = (0..10).map(|_| {
    let c = Arc::clone(&counter);
    thread::spawn(move || { *c.lock().unwrap() += 1; })
}).collect();
for h in handles { h.join().unwrap(); }
println!("{}", *counter.lock().unwrap());
```

❌ `Arc<RefCell<T>>` across threads — `RefCell` is `!Sync`, won't compile. `RefCell` is for single-threaded interior mutability; `Mutex`/`RwLock` for multi-threaded.

## 5. Rayon — data parallelism

**Rayon** is the dominant crate for data parallelism: turn `.iter()` into `.par_iter()` to parallelize a traversal across a thread pool. It uses work-stealing for load balancing.

✅ Trivially parallel map:
```rust
use rayon::prelude::*;
let doubled: Vec<i64> = (0..1_000_000)
    .into_par_iter()
    .map(|x| x * 2)
    .collect();
```

❌ Mutating shared state from inside `par_iter` without synchronization — either a borrow-checker error or a race if you reach for `unsafe`.

## 6. async/await and the async ecosystem

Rust's async/await returns a `Future` that is **inert** — it makes progress only when polled by an executor (runtime). This is unlike Go/Python; you must choose a runtime.

### Runtime alternatives — documented side by side

| Runtime | Characteristics |
|---|---|
| **Tokio** | The dominant async runtime. Multi-threaded work-stealing executor, full ecosystem (tokio-util, hyper, tonic, reqwest, sqlx all built on it). The de facto default for async Rust services. |
| **async-std** | Mirrors the std API with async versions; older alternative to Tokio. Less actively developed. |
| **smol** + **async-executor** | A small, modular async stack; popular for embedded/library use where you don't want to pull in Tokio. |
| **glommio** | Thread-per-core async for data-intensive workloads. |

> No runtime winner is picked here. Tokio is the most common choice for network services; smol for libraries that must stay runtime-agnostic. Document your choice in the README.

✅ Tokio service entrypoint:
```rust
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await?;
    loop {
        let (socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            // handle connection
        });
    }
}
```

## 7. `Pin` and `Unpin`

Async functions that hold references across `.await` points produce self-referential futures, which must be **pinned** in memory so their addresses stay stable. `Pin<P>` and the `Unpin` trait express this.

- Most types are `Unpin` (movable freely even when pinned).
- Self-referential futures are `!Unpin` and require pinning (`Box::pin`, or stack-pinning with `pin!`).

✅ Use `tokio::pin!` or `std::pin::pin!` (stable since 1.68) to pin a future on the stack:
```rust
let fut = async { /* ... */ };
std::pin::pin!(fut);
(&mut fut).await;
```

❌ Moving a `!Unpin` future after it has started being polled — memory unsafety.

## 8. The `futures` crate

The `futures` crate provides combinators (`FutureExt::map`, `StreamExt::next`, `Sink`, etc.) and the `Stream` trait (the async analog of `Iterator`). Streams are the idiomatic way to handle async sequences.

✅ Iterate an async stream:
```rust
use futures::StreamExt;
while let Some(item) = stream.next().await {
    handle(item);
}
```

## 9. Structured concurrency

Rust has no built-in structured-concurrency primitive yet (it's a known RFC discussion area), but the pattern is achieved by:
- `tokio::task::JoinSet` to spawn a group of tasks and await/abort them together.
- `tokio::select!` with a clear cancellation path.
- Crates like `tokio-util::task::JoinMap` and the experimental `tokio` task scopes.

✅ Use `JoinSet` to bound a group of tasks:
```rust
let mut set = tokio::task::JoinSet::new();
for i in 0..10 {
    set.spawn(async move { work(i).await });
}
while let Some(res) = set.join_next().await {
    // handle res
}
```

❌ Fire-and-forget `tokio::spawn` everywhere with no join handle — tasks can outlive their intent and leak resources.

## 10. Atomics and `Ordering`

`std::sync::atomic` provides `AtomicBool`, `AtomicUsize`, `AtomicU64`, `AtomicPtr`, etc. Pair them with the correct `Ordering`:

| Ordering | Use |
|---|---|
| `Relaxed` | Counters, stats — no ordering of other memory required. |
| `Acquire` (load) / `Release` (store) | Publishing data: store-release a flag after writing data, load-acquire to see the data. |
| `AcqRel` | Read-modify-write operations that need both. |
| `SeqCst` | Strongest, globally consistent; rarely needed and slowest. |

✅ Acquire/release pattern to publish data:
```rust
use std::sync::atomic::{AtomicBool, Ordering};
static READY: AtomicBool = AtomicBool::new(false);
// writer:
DATA.store(42, Ordering::Relaxed);
READY.store(true, Ordering::Release);
// reader:
while !READY.load(Ordering::Acquire) {}
assert_eq!(DATA.load(Ordering::Relaxed), 42);
```

❌ `Ordering::Relaxed` for a flag that gates access to non-atomic data — data race / torn read.

## 11. `tokio::spawn` vs `spawn_blocking`

A critical pitfall: **async tasks must not run long CPU-bound or blocking-IO code**, because they run on a small worker pool and blocking starves all other tasks on that thread.

- `tokio::spawn` — for async work (awaits, yields).
- `tokio::task::spawn_blocking` — for sync/blocking work (file IO, CPU-heavy computation, blocking FFI). Runs on a dedicated blocking thread pool.

✅ Offload blocking work:
```rust
let data = tokio::task::spawn_blocking(|| {
    std::fs::read_to_string("/big/file")   // blocking IO
}).await??;
```

❌ Calling `std::thread::sleep` or `std::fs::read` inside an `async fn` without `spawn_blocking` — stalls the runtime.

## 12. Common pitfalls

| Pitfall | Mitigation |
|---|---|
| **Deadlock** — acquiring locks in inconsistent order across threads. | Always acquire locks in a globally fixed order; prefer coarse locks or channels; consider lock-free structures. |
| **Blocking in async context.** | Use `spawn_blocking` for sync IO/CPU work; never call `std::thread::sleep`. |
| **Mutex poisoning** — a thread panicked while holding a std `Mutex`. | Handle `PoisonError` explicitly, or use `parking_lot` (no poisoning). |
| **Forgetting to await a future** (especially `tokio::spawn`'s `JoinHandle`) — task never runs or never observed. | `await` join handles, or use `JoinSet`. |
| **`Rc` across threads** — compile error. | Use `Arc`. |
| **Unbounded channels** — memory blowup if producer outruns consumer. | Prefer bounded channels (`mpsc::channel(N)`) to apply backpressure. |
| **`tokio::spawn` outside a runtime** — panic. | Ensure the runtime is running; `#[tokio::main]` at the entrypoint. |

## 13. Declaring thread-safety in docs

Convention: document the concurrency guarantees in rustdoc (`///`) — e.g., *"This type is `Send + Sync` and can be shared across threads via `Arc`."* or *"Not `Sync`; confine to one thread."* There is no formal `@threadsafe` tag; prose is the norm.

## Sources

| Topic | Source |
|---|---|
| The Rust Book, ch. 16 | https://doc.rust-lang.org/book/ch16-00-concurrency.html |
| Asynchronous Programming Book | https://rust-lang.github.io/async-book/ |
| `Send`/`Sync` | https://doc.rust-lang.org/nomicon/send-and-sync.html |
| Tokio | https://tokio.rs |
| async-std | https://async.rs |
| smol | https://github.com/smol-rs/smol |
| Rayon | https://github.com/rayon-rs/rayon |
| crossbeam | https://github.com/crossbeam-rs/crossbeam |
| flume | https://github.com/zesterer/flume |
| futures | https://github.com/rust-lang/futures-rs |
| parking_lot | https://github.com/Amanieu/parking_lot |
