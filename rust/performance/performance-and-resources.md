---
title: Rust - Performance and Resource Efficiency
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Performance and Resource Efficiency

> Rust's promise is **zero-cost abstractions** on top of fine-grained control over memory and CPU. This section catalogs the idiomatic performance patterns and the recognized profiling/benchmarking tools. There is no universal "optimize everything" rule — measure first (criterion, perf, flamegraph), optimize hot paths, and keep readability. Boundary: resource lifecycle in Rust is largely automatic via ownership/`Drop` (RAII); this section focuses on efficiency patterns. Canonical reference: The Rust Performance Book — https://nnethercote.github.io/perf-book/.

## 1. Zero-cost abstractions

Rust's iterators, closures, generics (monomorphization), and trait abstractions compile down to code as efficient as hand-written loops, because of inlining and monomorphization. You generally do not pay a runtime cost for using high-level constructs.

✅ Use iterators and combinators; the optimizer handles them:
```rust
let sum: i64 = numbers.iter().filter(|&&x| x > 0).map(|&x| x as i64).sum();
```

❌ Assuming the iterator version is slower and rewriting it as a manual `for` loop "for performance" — benchmark first; they are usually equal.

## 2. Iterator vs loop

Community consensus (and the perf-book): **iterators are usually as fast as loops** and often faster (autovectorization-friendly). Write the clear version; only deviate when a benchmark proves the loop wins on a hot path.

✅ Prefer the iterator chain for clarity:
```rust
let total: u64 = items.iter().map(|i| i.price).sum();
```

❌ Premature manual unrolling or `unsafe` to "speed up" a cold path.

## 3. Avoiding unnecessary allocations

Allocations (`Box`, `Vec`, `String`, `format!`) are the most common Rust perf cost. Idioms:
- Prefer `&str` over `String` for parameters that only need to read.
- Prefer `&[T]` over `&Vec<T>`.
- Borrow instead of `.clone()`.
- Use `Iterator` chains instead of collecting intermediate `Vec`s.

✅ Borrow and slice:
```rust
fn greet(name: &str) -> String {       // takes a borrow, returns owned
    format!("hello, {name}")
}
```

✅ Avoid an intermediate collection:
```rust
let count = iter.filter(|x| **x > 10).count();   // no Vec allocated
```

❌ Unnecessary cloning on a hot path:
```rust
fn process(items: &Vec<String>) {
    for s in items { let _ = s.clone(); }   // each clone allocates
}
```

❌ `format!` in a tight loop to build a log line that may never be emitted — use lazy formatting (` tracing::` macros defer this).

## 4. `Vec` pre-allocation — `Vec::with_capacity`

If you know (or can estimate) the final size, pre-allocate to avoid reallocation/copy cascades.

✅ Pre-allocate when size is known:
```rust
let mut out = Vec::with_capacity(inputs.len());
for x in inputs { out.push(transform(x)); }
```

Or, prefer collecting directly:
```rust
let out: Vec<_> = inputs.iter().map(transform).collect();
// collect() infers capacity from the iterator's size hint
```

❌ `let mut v = Vec::new();` followed by a known-count `push` loop with no `with_capacity` — triggers several reallocations.

## 5. `String` vs `&str`

- `&str` is a borrowed slice (pointer + length) — cheap, no allocation.
- `String` is an owned, growable heap buffer.
- Take `&str` for read-only parameters; return `String` only when you need ownership.

✅ `fn find(haystack: &str, needle: &str) -> Option<usize>`.

❌ `fn find(haystack: String, needle: String)` — forces every caller to allocate/own.

## 6. The cost of cloning

`.clone()` on `String`, `Vec`, `HashMap` is O(n) and allocates. On `Arc`/`Rc` it is cheap (refcount bump). On `Copy` types (`u32`, etc.) it is a bitwise copy.

✅ Use `Arc<T>` to share a heavy value cheaply across threads (clone the `Arc`, not the `T`).

❌ Cloning a `Vec<String>` of 10k elements on every call.

## 7. Smart pointers — `Box` / `Rc` / `Arc` (when each)

| Pointer | Counting | Thread-safe | Use when |
|---|---|---|---|
| `Box<T>` | unique | n/a (owned) | Heap-allocate a single owner; recursive types (`enum List { Cons(Box<Node>), Nil }`). |
| `Rc<T>` | single-threaded refcount | ❌ no | Share read-only data within one thread (cheap refcount). |
| `Arc<T>` | atomic refcount | ✅ yes | Share across threads. |
| `Weak<T>` / `Weak` | weak ref | varies | Break reference cycles (caches, parent back-links). |

✅ Use `Rc` only when you are certain of single-threaded use and the refcount overhead beats cloning; `Arc` when crossing threads.

❌ Using `Arc` where `Rc` would suffice — pays an atomic cost for no benefit. Conversely, `Rc` across threads is a compile error (it is not `Send`).

## 8. Branchless & cache-friendly patterns

- `Iterator::filter` and predicate logic can prevent autovectorization on hot loops; sometimes sorting data to be branch-predictable or using lookup tables helps.
- **Cache-friendly data layout**: `Vec<T>` (Array-of-Structs, AoS) is already cache-friendly; for very hot scans over one field, consider **Struct-of-Arrays (SoA)**.

### `smallvec` and friends
The **smallvec** crate stores a small number of elements inline (no heap allocation) until it spills — useful for collections that are usually tiny.

| Crate | Use |
|---|---|
| `smallvec` | Inline small vectors. |
| `tinyvec` | `#![no_std]`-friendly smallvec. |
| `arrayvec` | Fixed-capacity stack-backed array. |

✅ `SmallVec<[T; 8]>` for a collection that is empty/small in 95% of cases.

❌ `Vec` for a field that holds at most 4 bytes — pays a heap allocation per instance.

## 9. Benchmarking — `cargo bench` / criterion

The built-in `cargo bench` harness uses the unstable `#[bench]`; the **idiomatic replacement is the `criterion` crate**, which provides statistical benchmarking with outlier detection and regression comparisons.

✅ Criterion benchmark:
```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_sum(c: &mut Criterion) {
    let data: Vec<i64> = (0..1000).collect();
    c.bench_function("sum_iter", |b| {
        b.iter(|| data.iter().sum::<i64>())
    });
}

criterion_group!(benches, bench_sum);
criterion_main!(benches);
```

Run with `cargo bench`; criterion writes HTML reports under `target/criterion/` and supports regression checks against a baseline (`--save-baseline`, `--baseline`).

## 10. Profiling tools

| Tool | Role |
|---|---|
| **`perf`** (Linux) | CPU profiling, call graphs (`perf record`/`perf report`). |
| **`cargo flamegraph`** | Generates an SVG flamegraph via `perf`/`dtrace`. The easiest visual profiler. |
| **`samply`** | Modern statistical profiler with an excellent web UI; recommended for new projects. |
| **`dtrace`** (macOS/BSD) | System-wide profiling (used by flamegraph on mac). |
| **`hyperfine`** | Benchmark CLI execution time across versions/flags. |
| **`cargo instruments`** (macOS) | Apple Instruments integration. |

✅ `cargo flamegraph --bin myapp` for a fast first look at hot spots; `samply record ./myapp` for a modern interactive view.

❌ Optimizing without profiling — "I think this is slow" is wrong most of the time.

## 11. Profile-guided optimization (PGO)

**cargo-pgo** orchestrates PGO (and LLVM BOLT) for Rust: build an instrumented binary, run representative workload, then rebuild using the collected profile. Useful for release binaries where every percent counts.

```bash
cargo install cargo-pgo
cargo pgo run bench     # collect profile
cargo pgo build --release
```

## 12. Synchronization cost — `parking_lot`, atomics

| Primitive | Cost / use |
|---|---|
| `std::sync::Mutex` / `RwLock` | Correct, portable; some overhead (e.g., Rust's std `Mutex` is fair). |
| `parking_lot::Mutex` / `RwLock` | Faster, smaller, no poisoning; widely adopted. The recognized perf upgrade. |
| `AtomicBool/AtomicUsize/...` | Lock-free; correct with the right `Ordering`. Use for simple counters/flags. |
| `crossbeam` / `crossbeam-queue` | Lock-free/lock-based data structures (channels, queues, epoch-based reclamation). |

✅ Prefer atomics over a `Mutex` for a single counter:
```rust
use std::sync::atomic::{AtomicU64, Ordering};
static REQUESTS: AtomicU64 = AtomicU64::new(0);
REQUESTS.fetch_add(1, Ordering::Relaxed);
```

❌ Wrapping a single counter in a `Mutex` — orders of magnitude slower.

### `Ordering` reference
| Ordering | When |
|---|---|
| `Relaxed` | No ordering guarantees needed (counters). |
| `Acquire` / `Release` | Paired loads/stores guarding other memory. |
| `SeqCst` | Strongest; needed for some algorithms (costly). |

## 13. `Mutex` vs `RwLock`

- `Mutex<T>`: exclusive access; one writer/reader at a time.
- `RwLock<T>`: many readers **or** one writer. Use when reads vastly outnumber writes.

✅ `RwLock` for a config map that is read constantly and rarely updated.

❌ `RwLock` when writes are frequent — writer starvation and overhead beat a plain `Mutex`.

## 14. Resource lifecycle — RAII / `Drop`

Rust manages resources via **ownership and the `Drop` trait** (RAII). When a value goes out of scope, `drop` runs deterministically — closing files, sockets, locks, freeing memory. No GC pause, no `close()` to forget.

✅ Let RAII do its job:
```rust
{
    let f = std::fs::File::open("x")?;   // closed automatically at end of scope
    let _g = mutex.lock().unwrap();      // unlocked automatically
}
```

❌ Reimplementing `Drop` to call a "cleanup" function manually, or storing a raw handle and forgetting to close it.

## 15. Caching conventions

- Per-process in-memory caches: `moka` (high-performance, popular), `cached` (macro-based), `quick_cache`.
- Distributed caches: the usual Redis/Memcached clients (`redis`, `memcache`).
- Always decide cache **ownership**: which layer owns the cache and invalidation.

✅ `moka::Cache` for a concurrent LRU in a service; set `time_to_live` and `max_capacity`.

❌ A `static HashMap` with no eviction — unbounded memory growth.

## Sources

| Topic | Source |
|---|---|
| The Rust Performance Book | https://nnethercote.github.io/perf-book/ |
| criterion | https://github.com/bheisler/criterion.rs |
| cargo-flamegraph | https://github.com/flamegraph-rs/flamegraph |
| samply | https://github.com/mstange/samply |
| cargo-pgo | https://github.com/Kobzol/cargo-pgo |
| parking_lot | https://github.com/Amanieu/parking_lot |
| smallvec | https://github.com/servo/rust-smallvec |
| moka | https://github.com/moka-rs/moka |
