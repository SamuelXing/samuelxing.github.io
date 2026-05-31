---
title: 'Memory Ordering'
date: 2026-05-30
permalink: /posts/2026/05/memory-ordering-quick-notes/
tags:
  - rust
  - concurrency
  - atomics
---

Memory ordering is one of those topics where the first time you read about it, it sounds straightforward — "tell the compiler not to reorder things" — and then the second time you sit down to actually use it, you realize you don't quite know what you're doing. This is my attempt to write down the smallest mental model that still lets you reason correctly about lock-free code, with the examples that anchored each piece for me.

The whole subject only exists because of one inconvenient fact: the program you wrote is not the program your CPU runs. Compilers reorder instructions. CPUs reorder loads and stores. Caches don't synchronize between cores for free. In single-threaded code, none of this matters — the hardware guarantees the *observable* behavior matches your source. But the moment another thread is watching, the illusion breaks. Memory ordering is the vocabulary you use to claw back the guarantees you need, and *only* the guarantees you need.

## The thing that trips everyone up first

When people first encounter atomics, they tend to fuse two ideas that should stay separate:

1. **Atomicity** — does this operation happen in one indivisible step?
2. **Ordering** — what does this operation say about other memory around it?

These are genuinely independent properties. `fetch_add(1, Relaxed)` is atomic (the read-modify-write can't be torn) but says nothing about ordering. `load(Acquire)` is just a read (atomicity is trivial — it's one access) but emits a fence that controls what surrounding code can do. You can mix and match. The first big unlock is realizing they're two separate knobs.

If you only remember one thing from this section: *atomicity protects the variable itself; ordering protects everything else around it.*

## Relaxed: atomic, and nothing more

`Relaxed` gives you the weakest ordering — basically none. It promises only that the operation on this particular atomic happens indivisibly. It says nothing about other memory.

```rust
hit_count.fetch_add(1, Relaxed);     // counter is its own truth
if stop.load(Relaxed) { return; }    // flag is the only thing that matters
```

Both of these are fine. If two threads do `fetch_add(1, Relaxed)` on a counter holding `5`, you always end up with `7`, never `6` — that's atomicity, which Relaxed still gives you. The catch is anything *around* the atomic op is fair game for reordering.

That's why this code is broken:

```rust
PAYLOAD = 42;                        // (1) non-atomic write
READY.store(true, Relaxed);          // (2) BUG: reader may see READY=true with PAYLOAD still 0
```

The compiler or CPU is free to reorder (1) and (2), or to let (1) sit in a store buffer while another thread already sees (2). Relaxed gives you a coherent counter and nothing else. The moment you want one thread to *publish* something to another, you need a stronger ordering.

## Release and Acquire: the publication handshake

If Relaxed is the bare minimum, `Release` and `Acquire` are where most lock-free code actually lives. The intuition is publisher–subscriber:

- A **Release store** says: *everything I did before this is now published.*
- An **Acquire load** says: *whatever I do after this will see what was published.*

The pair only does its job if the Acquire actually reads the value the Release wrote (or something later). That's the handshake.

```rust
// Writer thread
PAYLOAD = 42;                        // (1)
READY.store(true, Release);          // (2) publishes (1)

// Reader thread
while !READY.load(Acquire) {}        // (3) pairs with (2) once it sees `true`
let v = PAYLOAD;                     // (4) guaranteed to see 42
```

What's elegant about this is that one Release publishes *everything* — every non-atomic write, every Relaxed atomic, every heap mutation that happened before it. You don't need to mark each individual write. That's why a typical lock-free data structure uses Relaxed for the bulk of its internal state and a single Release on the flag or tail pointer that "commits" the update.

The mnemonic I keep in my head: **Release pins the past, Acquire pins the future.** Release won't let prior writes slide forward past it; Acquire won't let later reads slide backward past it.

## Fences are one-way, and that's the whole trick

This part took me a while to internalize, but it's where the design becomes elegant rather than arbitrary. An Acquire or Release isn't a "full barrier" that blocks everything in both directions. Each is a **one-way fence** — it only blocks motion in the direction that would break the handshake.

```
  ↑ later code
  │
  ─── Acquire load ───      ← later code can NOT cross upward past Acquire
  ↓ earlier code            ← earlier code CAN sink downward past Acquire


  ↑ later code              ← later code CAN rise upward past Release
  ─── Release store ───     ← earlier code can NOT cross downward past Release
  ↓ earlier code
```

Why one direction each? Because that's all you need. The publisher's job is to make sure *prior* writes don't slip below the Release — that's what makes them visible to the subscriber. Anything *after* the Release is irrelevant; let it move freely. Symmetrically, the subscriber's job is to make sure *later* reads don't slip above the Acquire — that's what makes them see the published values. Anything *before* the Acquire is its own story.

This asymmetry is why Acquire/Release is cheaper than a full barrier. Each side blocks only the reordering that would actually break the handshake. Everything else, the hardware gets to keep doing for performance.

The handshake spelled out one more time, slowly: every write that happened *before* the Release on thread A becomes visible *after* the matching Acquire on thread B, *if* B's Acquire reads the value A's Release wrote.

### Standalone fences

There's also `std::sync::atomic::fence(ordering)` — a barrier that isn't attached to a specific atomic operation. Same one-way semantics, just detached. It's useful when you want to upgrade ordering conditionally — pay for the fence only on the path that actually consumed the value.

```rust
let v = flag.load(Relaxed);          // cheap check
if v {
    fence(Acquire);                  // upgrade to Acquire only when it matters
    let payload = PAYLOAD;           // now sees what the writer published
}
```

Most code doesn't need this. Prefer ordering on the access itself unless profiling tells you the unconditional Acquire is showing up in a hot loop.

## AcqRel: the same handshake, but fused

Once you understand Acquire and Release, AcqRel is straightforward — it's both, on a single read-modify-write. The reason it exists isn't ordering (you could get that from a separate load and store), but **atomicity**. Splitting the RMW into two steps lets another thread interleave:

```rust
// WRONG — two ops, race window between them
let cur = counter.load(Acquire);     // reads 5
                                     // ← another thread increments to 6
counter.store(cur + 1, Release);     // writes 6 — lost update

// RIGHT — one indivisible RMW
counter.fetch_add(1, AcqRel);
```

So when you reach for AcqRel, you're saying two things at once: *I need to see what prior publishers did* (Acquire), *and I need to publish my own work* (Release), *and these two things must happen as one step so no one can sneak in between* (atomicity). That combination shows up constantly in real lock-free code: `Arc::drop`, CAS loops on the head of a lock-free stack, hand-off counters where each participant both reads the running total and contributes to it.

## SeqCst: when pairwise isn't enough

For most code, Acquire/Release is the right tool. But there's a class of problems where it isn't strong enough, and that's where `SeqCst` (sequentially consistent) earns its keep.

The shape that breaks Acquire/Release is when you have *independent* atomics and you need observers to agree on the order between them. The canonical case is mutual flagging — two threads each raise their own flag and check the other:

```rust
// Thread A                          // Thread B
A.store(true, SeqCst);               B.store(true, SeqCst);
if B.load(SeqCst) { /* saw B */ }    if A.load(SeqCst) { /* saw A */ }
```

You want the guarantee that at least one thread sees the other's flag. With plain Release/Acquire, this can fail — the Release pins prior writes, but it doesn't stop the *later* load from drifting up past the store. Both threads can do their store, both can do their load before the other store is visible, and both miss each other.

SeqCst fixes this by establishing a single global total order over every SeqCst operation in the program. Every thread agrees on that order. So at least one thread's load is going to land *after* both stores, and that thread sees the other's flag.

The cost: a real full barrier (`MFENCE` on x86, dmb ish on ARM). Reach for SeqCst when you have this specific shape — independent atomics where observers must agree on order — and use the weaker tools otherwise.

## Picking one

The mental flow I actually use, in order of how often each step matters:

```
Pure load (no modify)?
  └─ Just need atomicity?           → Relaxed
  └─ Reading published data?        → Acquire

Pure store (no read)?
  └─ Just bumping a flag/value?     → Relaxed
  └─ Publishing data for readers?   → Release

RMW (fetch_*, CAS, swap)?
  └─ Counter / stat only?           → Relaxed
  └─ Publish, don't need to see?    → Release
  └─ See, don't need to publish?    → Acquire
  └─ Both publish AND see?          → AcqRel
  └─ Need cross-atomic agreement?   → SeqCst
```

The rule underneath the flow: pick the weakest ordering that gives you the happens-before relation you actually need. Anything stronger is paying for synchronization you aren't using — and on weakly-ordered architectures (ARM, RISC-V), that cost is real, not theoretical.

## If you only remember a few things

**Relaxed** is just atomicity on one variable — every atomic is its own little universe with no relation to the rest.

**Acquire and Release** are a pairwise handshake. The publisher's prior work becomes visible to whoever reads what the publisher wrote. Each side is a one-way fence — only the direction that matters for the handshake is blocked.

**AcqRel** is the same handshake fused into one indivisible RMW, so nothing can slip between seeing and publishing.

**SeqCst** is the handshake plus a single global timeline that every thread agrees on. You need it less often than you think.

The ladder is Relaxed → Acquire/Release (with AcqRel as its fused RMW form) → SeqCst, going from cheapest and weakest to most expensive and strongest. Most lock-free code lives on the middle rung. If you find yourself reaching for SeqCst, it's worth pausing to ask whether there's an Acquire/Release shape that would do the same job.
