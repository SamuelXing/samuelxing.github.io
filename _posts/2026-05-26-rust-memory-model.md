---
title: 'Rust Memory Model: A Practical Guide to Atomic Orderings'
date: 2026-05-26
permalink: /posts/2026/05/rust-memory-model/
tags:
  - rust
  - concurrency
  - atomics
  - memory-model
  - ai-teach-me-something
---

A working reference for the four atomic orderings in Rust — Relaxed, Acquire/Release, AcqRel, and SeqCst — built around the patterns where each one is the right choice (and the misconceptions that bite when it isn't).

## The two-axis mental model

Atomic operations have **two independent properties**:

| Axis | What it controls | Set by |
|---|---|---|
| **Operation atomicity** | Whether load and store happen as one indivisible step | The method name (`load`, `store`, `fetch_add`, `swap`, `compare_exchange`, ...) |
| **Memory ordering** | Which fences are emitted, what happens-before is established | The `Ordering` parameter |

These are independent. `fetch_add(Relaxed)` is *atomic* (indivisible load+modify+store) but emits *no fences*. `load(Acquire)` is *not an RMW* (it's pure read) but emits an *Acquire fence*. Conflating the two axes is the source of most memory-ordering confusion.

Rows = the **fences** emitted (memory ordering). Columns = whether the operation itself is **atomic** (a single indivisible RMW) or **non-atomic** (a separate load and store that another thread can slip between).

| Fence (rows) ↓ \ Atomicity (cols) → | **Non-atomic** — separate `load` + `store` | **Atomic** — single RMW (`fetch_add`, `swap`, `compare_exchange`, …) |
|---|---|---|
| **Relaxed** — no fence | `load(Relaxed)` / `store(Relaxed)` | `fetch_add(Relaxed)` |
| **Acquire** — fence on loads only | `load(Acquire)` *(no `store(Acquire)` exists)* | `fetch_add(Acquire)` |
| **Release** — fence on stores only | `store(Release)` *(no `load(Release)` exists)* | `fetch_add(Release)` |
| **Acq + Rel** — both halves | `load(Acquire)` + `store(Release)` — **race-prone**: another thread can write between the load and the store | `fetch_add(AcqRel)` — **indivisible**: no window for another writer |
| **SeqCst** — Acq+Rel **plus global total order** across every SeqCst op in the program | `load(SeqCst)` + `store(SeqCst)` — still two ops, but participate in the global order | `fetch_add(SeqCst)` — atomic *and* in the global order |

**How to read the columns.**

- Pick the **row** by the synchronization (fence) you need.
- Pick the **column** by whether you need to read-modify-write in one step.

**The punchline is the Acq+Rel row.** `load(Acquire)` followed by `store(Release)` has the same fence strength as `fetch_add(AcqRel)` — but it is *two* operations, so it loses updates under contention. `fetch_add(AcqRel)` fuses them. That's the entire reason RMW operations exist.

**The `—` cells aren't missing — they don't exist.** `Acquire` is defined on loads, `Release` on stores. There is no `store(Acquire)` or `load(Release)` in the API.

---

## The ordering ladder

```
  Relaxed                        no synchronization for OTHER memory
     │
     │ + per-atomic pairwise happens-before
     ▼
  Acquire/Release (split)        pairwise sync via this atomic   ──┐
     │                                                              │ SAME
     │                                                              │ sync
     ▼                                                              │ power
  AcqRel (fused into one RMW)    pairwise sync + RMW atomicity   ──┘
     │
     │ + GLOBAL agreement across ALL SeqCst ops on different atomics
     ▼
  SeqCst
```

**Key insight**: AcqRel and Acquire/Release sit on the same row for synchronization strength. AcqRel doesn't climb toward SeqCst — it just packages the two halves into one indivisible RMW. The jump to SeqCst is qualitatively different: it adds *global agreement* across multiple atomics.

---

## Relaxed — atomic, but each atomic is its own universe

### What it guarantees

1. **Atomicity of the single access** — no torn reads/writes.
2. **Per-atomic modification order** — all writes to one atomic happen in some total order, all threads agree on it.
3. **Per-thread, per-atomic source order** — your own ops on the same atomic appear in source order to all observers.

### What it does NOT guarantee

1. **Cross-atomic ordering** — operations on different atomics can be observed in any order from other threads.
2. **Happens-before with non-atomic memory** — reading a value tells you nothing about other memory the writer may have touched.

### When Relaxed is correct

- **Counters**: `hit_count.fetch_add(1, Relaxed)`. No other memory depends on the counter's specific value at a specific moment.
- **Stop flags** where the receiver only reads the flag: `if stop.load(Relaxed) { return; }`.
- **Lazy init of pure functions** where all writers compute the same value.
- **Progress reporting**: writer increments a counter; reader samples it.

### Common misconception

> "Relaxed means not atomic, so two threads might lose an update."

**Wrong.** `fetch_add(Relaxed)` is fully atomic. Two threads doing `fetch_add(1, Relaxed)` on a counter starting at 5 will produce 7 — never 6. Atomicity is a property of the operation, not the ordering.

What Relaxed gives up is publication of *other* memory, not the atomicity of the operation itself.

---

## The publication bug — why Relaxed isn't enough for shared structures

The classic anti-pattern:

```rust
static mut PAYLOAD: u64 = 0;
static READY: AtomicBool = AtomicBool::new(false);

// Writer:
unsafe { PAYLOAD = 42; }           // (1) non-atomic write
READY.store(true, Relaxed);        // (2) Relaxed publish — WRONG

// Reader:
while !READY.load(Relaxed) {}      // (3) Relaxed acquire — WRONG
let v = unsafe { PAYLOAD };        // (4) may read 0 (garbage)
```

With Relaxed on both sides:
- The compiler or CPU may reorder (1) and (2) — the reader sees READY=true with PAYLOAD still 0.
- Even with (1) and (2) in order, the compiler may hoist (4) above the loop on (3).

The fix is Release/Acquire on the synchronizing flag.

---

## Release / Acquire — the one-sided fences

### The plain-English rule

- **Release** = "I'm publishing my work."  → Anything I did *before* this point must finish before the Release.
- **Acquire** = "I'm about to use what was published."  → Anything I do *after* this point must happen after the Acquire.

**Mnemonic: Release pins the past. Acquire pins the future.**

### The fence diagram

```
═══ Release ═══              ═══ Acquire ═══

write A  ──┐                 read X  ──┐
write B  ──┼─ PINNED ABOVE   read Y  ──┼─ free to drift DOWN
write C  ──┘                 read Z  ──┘
─────────────                ─────────────
RELEASE store                ACQUIRE load
─────────────                ─────────────
write D  ──┐                 read A  ──┐
write E  ──┼─ free to drift  read B  ──┼─ PINNED BELOW
write F  ──┘  UP             read C  ──┘
```

"Drift" means crossing the fence. Release lets later writes drift up (cross to before the fence). Acquire lets earlier reads drift down (cross to after the fence). Each fence blocks crossing in one direction and allows it in the other.

### Same-side reordering is unconstrained

Within "before the fence" (or "after the fence"), Release/Acquire add **no** order between members. A, B, C can be reordered freely among themselves — Release only constrains crossings.

```
Source order:                Valid reordering:
─────────────                ─────────────────
write A                      write C    ← all three permuted
write B                      write A
write C                      write B
═══ RELEASE ═══              ═══ RELEASE ═══
```

Other rules (data dependencies, the as-if rule, atomic-on-atomic ordering) may still constrain things — but Release itself doesn't.

### Why the asymmetric drift is harmless

- Release lets later writes drift up: the reader's Acquire only synchronizes with what was visible *at* the Release. Later writes weren't promised, so floating them up just makes them visible too — never invisible.
- Acquire lets earlier reads drift down: those reads weren't synchronized with the publisher anyway (they happened before the Acquire claimed sync). Shifting them down doesn't change what they read.

### The fix to the publication bug

```rust
// Writer:
unsafe { PAYLOAD = 42; }           // (1)
READY.store(true, Release);        // (2) Release publishes (1)

// Reader:
while !READY.load(Acquire) {}      // (3) Acquire pairs with (2)
let v = unsafe { PAYLOAD };        // (4) guaranteed to see 42
```

The synchronization is **conditional**: it only establishes happens-before *if the Acquire load actually reads the value the Release store wrote* (or a later one in modification order). That's why publication loops `while !flag.load(Acquire) {}` instead of doing a one-shot check.

### What gets published

A Release publishes *everything* the writer did before it — in one shot:

- Non-atomic writes (the `PAYLOAD = 42`).
- Other atomic writes — even **Relaxed** ones.
- Heap allocations (`Box::leak(...)` pointers; the heap bytes become visible).
- Mutations to Vec/HashMap/etc.
- Transitively, anything the writer itself acquired earlier.

This is why most lock-free code uses Relaxed for most atomics and just one Release on the "publish" flag. The Release is doing all the synchronization work; everything else rides along.

### Where this exact pattern lives

Every concurrent primitive in `std` reduces to Release/Acquire:

| Primitive | Release point | Acquire point |
|---|---|---|
| Mutex | `unlock` | `lock` |
| Arc | each `drop` | last `drop`'s fence |
| OnceLock | `set` | `get` |
| channel | `send` | `recv` |
| RCU | pointer swap | pointer read |
| Future::poll signaling | waker drop | next poll's load |

Strip the orderings and any of these become unsound.

---

## SeqCst — global agreement across multiple atomics

### What SeqCst adds

There exists **one global total order** over all SeqCst operations across all threads, and every thread agrees on that order. Acquire/Release only makes *pairwise* promises — two observers of unrelated events can disagree on their order.

### Two-thread, two-observer pattern (IRIW)

```
Thread 1:  x.store(true)        // independent
Thread 2:  y.store(true)        // independent
Thread 3:  load x → ?,  load y → ?
Thread 4:  load y → ?,  load x → ?
```

**Under Acquire/Release:**
```
   Thread 1  ──sync──►  Thread 3:  x=true, y=false   ("x came first")
   Thread 2  ──sync──►  Thread 4:  y=true, x=false   ("y came first")
```
Two pairwise chains, no cross-edges. Observers can legally disagree. Both views are consistent with what the synchronizes-with graph permits.

**Under SeqCst:**

```
Global timeline (everyone sees this):
─●─────────●─────────●─────────●─────────●─►
 ↑         ↑         ↑         ↑         ↑
 x.store   y.store   T3.x.load T3.y.load T4.x.load ...
```

One picked order. If `x.store` is before `y.store` in the timeline, all observers see "x came first." Disagreement is impossible.

### The Dekker pattern — only SeqCst suffices

```rust
// Thread A:                       // Thread B:
A.store(true, ???);                B.store(true, ???);
if B.load(???) { /* saw B */ }     if A.load(???) { /* saw A */ }
```

Can both threads miss each other?

**With Release/Acquire**: yes. Release pins prior *writes*, not later *loads*. The load below the Release can drift up (or, equivalently on x86, the store stays in the store buffer while the load reads). Both threads can effectively load-before-store and both miss.

**With SeqCst**: no. Both stores and both loads are in one global timeline. Each thread's own store must come before its own load. In every valid global ordering, at least one load lands after both stores — that load sees the other thread's flag.

### SeqCst doesn't pick the order; it picks ONE order all threads agree on

A common confusion: "if SeqCst gives a total order, which order is it?"

It's whatever order the universe picks on this run. Different runs may interleave differently. SeqCst's promise isn't *which* order — it's *consistency*: all threads see the same one.

```
What SeqCst FORBIDS:                What SeqCst ALLOWS:

Observer P:  A,B,C,D                run 1:  A,C,B,D  ← all agree
Observer Q:  C,A,D,B  ✗             run 2:  C,A,B,D  ← all agree
                                    run 3:  A,B,C,D  ← all agree
```

Concurrent code must be correct under *any* valid interleaving. The Dekker proof works because the set of valid SeqCst interleavings excludes the "both miss" interleaving — but it includes many others.

---

## AcqRel — Acquire + Release fused into one RMW

### What it is

AcqRel is the ordering for RMW operations (`fetch_*`, `swap`, `compare_exchange`) when you need BOTH:
- the **load half** to be Acquire (see prior publishers' work)
- the **store half** to be Release (publish your work to future loaders)

Pure loads can't be Release (nothing to publish). Pure stores can't be Acquire (nothing to receive). Only RMWs do both, so they're the only ops where AcqRel applies.

### The unique capability AcqRel has over split Acq/Rel

A separate `load(Acquire) + store(Release)` looks equivalent on paper, but **another thread can interleave between them**:

```rust
// WRONG — race condition
let cur = counter.load(Acquire);         // Thread A reads 5
                                         // ◄── Thread B sneaks in, increments to 6
counter.store(cur + 1, Release);         // Thread A writes 6 (lost update!)
```

`fetch_add(1, AcqRel)` does the same thing atomically — no interleaving possible:

```rust
counter.fetch_add(1, AcqRel);            // 5 → 6, indivisibly
                                         // Thread B: 6 → 7, indivisibly
```

So:
- **Fence semantics**: AcqRel = split Acq/Rel (same memory ordering effect).
- **Operation atomicity**: AcqRel = atomic (no interleaving). Split Acq/Rel = race-prone.

### Where AcqRel earns its keep

- **Arc::drop**: `fetch_sub` publishes prior writes AND, on the last drop, needs to acquire everyone else's writes before dealloc.
- **Lock-free stack push**: CAS publishes new node's contents AND acquires the head pointer's prior writers' work.
- **Hand-off counters**: each contributor sees all prior contributors' work AND publishes its own.

### AcqRel does NOT fix Dekker

Even if you upgrade Dekker's stores to AcqRel CAS, both threads can still miss each other. AcqRel only establishes pairwise sync via *this atomic* — it says nothing about the relationship between different atomics. Only SeqCst's global timeline rules out the "both miss" case.

---

## The wrong orderings — what's forbidden

Rust enforces semantic constraints at the API level (runtime panics; `clippy::invalid_atomic_ordering` catches them at lint time):

| Operation | Allowed orderings | Forbidden orderings |
|---|---|---|
| `load` | Relaxed, Acquire, SeqCst | **Release, AcqRel** |
| `store` | Relaxed, Release, SeqCst | **Acquire, AcqRel** |
| `fetch_*` / RMW | Relaxed, Acquire, Release, AcqRel, SeqCst | (none) |
| `fence` | Acquire, Release, AcqRel, SeqCst | Relaxed |

### Why these are semantically meaningless

- `store(Acquire)` would mean "a write that *receives* publication." But a write doesn't read; there's nothing to receive.
- `load(Release)` would mean "a read that *publishes* prior work." But a read doesn't carry a payload to publish.

The asymmetry of Acquire (receive) and Release (publish) is baked into what each one means. A pure load has only the receive side; a pure store has only the publish side. AcqRel exists *because* RMWs are the one operation type that does both in one step.

---

## Misconceptions worth retiring

**"Relaxed is faster."**  
Relaxed has no extra cost on x86 (Acquire/Release are free there too). On ARM, yes — Relaxed avoids barriers. But "Relaxed for performance" is the wrong instinct: pick Relaxed when no other memory depends on the atomic. It's a correctness choice that happens to be free.

**"SeqCst is safer, just use it everywhere."**  
SeqCst masks bugs that weaker orderings would expose during testing, costs barriers on ARM/POWER, and `MFENCE` (~30 cycles) on x86. Pick the weakest ordering that establishes the happens-before you need.

**"Seeing X = 5 means I'm synchronized with whoever wrote 5."**  
Only if BOTH sides used Acquire/Release (or SeqCst) on the same atomic. A Relaxed read of 5 tells you nothing about other memory.

**"Memory ordering applies to a single atomic."**  
It's about the ordering of *other* accesses around this atomic. The atomic itself is always atomic; ordering controls what surrounds it.

**"AcqRel is stronger than Acq/Rel."**  
Same fence semantics. The difference is that AcqRel applies to an RMW (atomic load+store) whereas split Acq/Rel is two separate operations (race-prone).

**"AcqRel and SeqCst are interchangeable for RMWs."**  
AcqRel gives pairwise sync. SeqCst gives global agreement across atomics. Use AcqRel unless you have a Dekker-shaped pattern requiring cross-atomic consensus.

**"Per-atomic modification order requires SeqCst."**  
Even Relaxed has it. Every atomic has a per-atomic total order that all threads agree on, regardless of ordering. SeqCst's contribution is global agreement *across* atomics.

---

## Cost on real hardware

| Operation | x86_64 | AArch64 |
|---|---|---|
| `load(Relaxed)` | plain `MOV` | plain `LDR` |
| `load(Acquire)` | plain `MOV` (free) | `LDAR` (one extra cycle) |
| `store(Relaxed)` | plain `MOV` | plain `STR` |
| `store(Release)` | plain `MOV` (free) | `STLR` (one extra cycle) |
| `load(SeqCst)` | plain `MOV` | `LDAR` |
| `store(SeqCst)` | `XCHG` (implicit LOCK) | `STLR + DMB` |
| `fetch_add(Relaxed)` | `LOCK XADD` | `LDXR/STXR` loop (or LSE `LDADD`) |
| `fetch_add(SeqCst)` | `LOCK XADD` | `LDAXR/STLXR + DMB` |
| `fence(SeqCst)` | `MFENCE` (~30 cycles) | `DMB ISH` |

**x86 takeaway**: Acquire/Release are free on plain loads/stores. SeqCst stores and fences cost noticeably more.

**ARM takeaway**: every ordering above Relaxed costs an instruction-level barrier. Picking Acq/Rel over SeqCst roughly halves the cost; Relaxed eliminates it.

---

## Decision framework

```
What does your atomic operation do?
├─ Pure load           → Acquire (if synchronizing) or Relaxed
├─ Pure store          → Release (if publishing) or Relaxed
└─ RMW (fetch_*, CAS, swap)
   ├─ Counter / stat (no other memory matters)     → Relaxed
   ├─ Publishing only (no need to receive)         → Release
   ├─ Receiving only (no need to publish)          → Acquire
   ├─ Both publish AND receive in one RMW          → AcqRel
   └─ Cross-atomic global agreement (Dekker-like)  → SeqCst
```

### Interview cheat sheet

1. **State the data first** — pick the shape of shared state (bit, counter, value, queue, latest-only) before picking the primitive.
2. **Pick the weakest ordering** that establishes the happens-before you need.
3. **Default to Acquire/Release** for publication patterns.
4. **Default to AcqRel** for RMWs that both contribute and observe.
5. **Reach for SeqCst only** when you have multi-atomic agreement requirements (the Dekker shape).
6. **Use Relaxed** when no other memory depends on the atomic.

---

## Worked example: the Treiber stack (push)

```rust
use std::ptr;
use std::sync::atomic::{AtomicPtr, Ordering::Acquire, Ordering::AcqRel};

struct Node {
    value: u32,
    next: *mut Node,
}

fn push(head: &AtomicPtr<Node>, value: u32) {
    let node = Box::into_raw(Box::new(Node { value, next: ptr::null_mut() }));
    loop {
        let cur = head.load(Acquire);
        unsafe { (*node).next = cur; }
        // AcqRel on the CAS:
        //   Acquire half: see writes from the previous successful pusher
        //   Release half: publish *node's contents to future Acquire-loaders
        match head.compare_exchange_weak(cur, node, AcqRel, Acquire) {
            Ok(_)  => return,
            Err(_) => continue,
        }
    }
}
```

Why each ordering:
- `head.load(Acquire)`: see whatever the previous successful pusher wrote into their node's `next` and `value`.
- `compare_exchange_weak(.., AcqRel, Acquire)`:
  - **Success ordering AcqRel**: Acquire half receives the prior pusher's publication; Release half publishes *our* node's contents (the `next` pointer and `value` we just wrote) to anyone who later acquires `head`.
  - **Failure ordering Acquire**: on failure, the CAS degenerates to a load — we still want Acquire so the loop can correctly see the actual current value.

Why not SeqCst? No multi-atomic agreement needed. Only `head` matters. AcqRel gives the right pairwise sync; SeqCst would just add cost.

Why not split Acq/Rel? Can't — we need the load and store to be atomic so two pushers don't both succeed on a stale `cur`.

---

## Worked example: counter that gates data

```rust
use std::sync::atomic::{AtomicU64, Ordering::AcqRel, Ordering::Acquire, Ordering::Relaxed};

// Each contributor writes its data first, then increments the counter
// with AcqRel to BOTH publish its data AND see prior contributors' data.

fn contribute(counter: &AtomicU64, my_data: &AtomicU64, my_value: u64) -> u64 {
    my_data.store(my_value, Relaxed);   // write our payload
    counter.fetch_add(1, AcqRel)        // Acquire prior + Release ours
}

// Reader can iterate over contributors' data — all prior contributors'
// writes are visible because each fetch_add's Acquire half pulled them in.
```

Why AcqRel: the counter is both the publish point (we contribute) and the receive point (we observe prior contributions). One atomic RMW serves both roles.

---

## Worked example: Arc-style refcount drop

```rust
use std::sync::atomic::{fence, AtomicUsize, Ordering::Acquire, Ordering::Release};

struct ArcInner<T> {
    count: AtomicUsize,
    data: T,
}

fn drop_arc<T>(inner: *const ArcInner<T>) {
    let inner = unsafe { &*inner };
    if inner.count.fetch_sub(1, Release) != 1 {
        return;                  // not the last, just published our writes
    }
    // We're the last. Acquire fence to see EVERYONE's prior Released writes
    // before we deallocate.
    fence(Acquire);
    // ... deallocate ...
}
```

Why Release on `fetch_sub`: every dropper must publish its prior writes to *T*, in case it was the last one mutating it.

Why a separate Acquire fence (not `AcqRel` on the `fetch_sub`): the Acquire is only needed on the last drop. Paying for it on every drop (via AcqRel) costs perf on the common path. Splitting saves the Acquire for when we actually need it.

This is the pattern in the real `std::sync::Arc` and is worth internalizing — it shows the value of weakening to Release-only when Acquire isn't always needed.

---

## The one-line distillations

> **Relaxed**: atomic per access, but each atomic is its own universe.
> **Acquire/Release**: pairwise handshake — publisher promises "my prior work is now visible to anyone who reads what I wrote."
> **AcqRel**: same handshake but fused into one indivisible RMW so other threads can't interleave between the load and store.
> **SeqCst**: pairwise handshake PLUS a global timeline that all threads agree on, across all SeqCst ops on all atomics.

The progression is: Relaxed (no sync) → Acq/Rel and AcqRel (pairwise sync, two packagings) → SeqCst (global sync). Pick the weakest level that establishes the happens-before your correctness actually needs.

---

## Further reading

- *Rust Atomics and Locks* by Mara Bos — https://marabos.nl/atomics/
- Rust standard library docs on `std::sync::atomic::Ordering`
- C++20 memory model — the same model formally; reading their spec helps even if you write Rust.
- The `loom` crate — model-checks concurrent Rust code by exploring all valid memory-model interleavings.
- The `atomic-wait` crate — futex abstraction for building higher-level primitives.
