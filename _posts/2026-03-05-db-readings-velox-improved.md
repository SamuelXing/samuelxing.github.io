---
title: 'Velox: Meta''s Unified Execution Engine(2022)'
date: 2026-03-08
published: true
tags:
  - databases
  - query processing
---

Velox addresses the massive duplication of effort across the data industry, where every query engine (Spark, Presto, Flink, etc.) independently implements the same core primitives — type systems, memory management, expression evaluation, vectorized execution — with inconsistent quality. By extracting these into a shared, open-source C++ library, Velox allows engine builders to focus on their differentiating features (SQL dialect, scheduling, optimizer) rather than re-implementing low-level execution from scratch.

Link: [p3372-pedreira.pdf](https://15721.courses.cs.cmu.edu/spring2024/papers/05-execution2/p3372-pedreira.pdf)

<details markdown="1">
<summary><b>Detailed Notes</b></summary>

### Why is this paper important? (1-2 sentences)
Velox addresses the massive duplication of effort across the data industry, where every query engine (Spark, Presto, Flink, etc.) independently implements the same core primitives — type systems, memory management, expression evaluation, vectorized execution — with inconsistent quality. By extracting these into a shared, open-source C++ library, Velox allows engine builders to focus on their differentiating features (SQL dialect, scheduling, optimizer) rather than re-implementing low-level execution from scratch.

### What are the key ideas/main contributions? (2-3 sentences)
Velox provides a modular execution library (not a standalone engine) with: a rich type system, Arrow-compatible columnar in-memory format (Vectors), a vectorized expression evaluator with adaptive optimization (common subexpression elimination, constant folding, dictionary-aware processing), and a set of relational operators (filter, project, hash join, hash aggregate, etc.). A key architectural decision is the plugin-based extensibility model — users register custom types, functions, operators, and I/O connectors without modifying Velox internals. The adaptive execution features (runtime filter pushdown, dynamic filter reordering based on selectivity) go beyond static optimization.


### What system was built/modified, and how was it evaluated? (1-2 sentences)
Velox is deployed at Meta as the execution backend for Presto (via Prestissimo), Spark, and several internal streaming/ML systems. On TPC-H benchmarks, Prestissimo (Velox-backed) achieved 6-8x speedup over Presto's Java engine, demonstrating that native vectorized execution in a shared library can match or exceed purpose-built engines.


### Judgement and follow-up ideas or open questions? (1-2 sentences)
The "shared library" approach is compelling but faces an adoption challenge: every engine that integrates Velox must adapt its plan representation and semantics to Velox's model, which is non-trivial. The deeper question is whether one execution layer can truly serve OLAP, streaming, and ML workloads without becoming bloated, or whether the abstraction leaks under divergent performance requirements.


### Deep Dive: Memory Pool

#### Why not just use malloc/new?
A database engine needs a *memory manager*, not just a *memory allocator*. `malloc` has no accounting (can't track per-query/operator usage), no graceful degradation (OOM = crash), accumulates fragmentation over time, has no hierarchy or policy, and is optimized for many small allocations — the opposite of database patterns (few large, batch-lifetime allocations like hash tables and sort buffers).

#### How Velox achieves zero fragmentation: mmap + madvise
Velox bypasses malloc and manages memory at OS page level:
1. **`mmap`** reserves a large contiguous virtual address range (no physical RAM allocated yet)
2. Pages are faulted in on demand when touched
3. **`madvise(MADV_DONTNEED)`** releases physical pages back to the OS *without* releasing the virtual address range
4. Re-accessing those addresses faults in fresh pages — no fragmentation because the virtual space stays contiguous

#### Hierarchical memory pools
```
Root MemoryPool (system-wide limit, e.g. 64GB)
├── Query A Pool (limit: 10GB)
│   ├── HashJoin operator (3GB)
│   └── Aggregate operator (2GB)
└── Query B Pool (limit: 10GB)
    └── TableScan operator (1GB)
```

#### Overflow handling: arbitration
When an operator exceeds its limit, the **memory arbitrator** kicks in:
1. First, try to reclaim from the **same query** — spill the cheapest operator (Sort is cheapest, then Aggregate, then HashJoin)
2. If not enough, reclaim from **other queries** by forcing their operators to spill
3. If nothing can be spilled, queue the request or fail the query gracefully — never crash

Operators implement a `reclaimableBytes()` / `reclaim()` interface. They don't decide *when* to spill — the arbitrator makes the global decision based on spill cost.

#### Memory Pool vs Buffer Pool

|  | Buffer Pool (traditional DBMS) | Memory Pool (Velox) |
|---|---|---|
| **Role** | Cache for disk pages | Allocator for execution state |
| **Unit** | Fixed-size pages (4KB/8KB) | Variable-size (vectors, hash tables) |
| **Disk** | Fundamental — every page has disk backing | Optional — spill only under pressure |
| **Stores** | Persistent table/index data | Ephemeral intermediate query state |
| **Eviction** | Always happening (LRU/clock) | Only under memory pressure |

Velox has no buffer pool because it's an execution engine, not a storage engine. Data comes from external sources via I/O connectors; the storage layer handles its own caching.

### Evaluate my original writeup

| Aspect | Original | How to improve |
|---|---|---|
| **Importance** | Correctly identifies the problem (many specialized engines) and solution (unified reusable component). | Emphasize *why* it matters: duplicated engines mean duplicated bugs, inconsistent semantics, and wasted engineering effort. |
| **Key ideas** | "Reusable, extensible and high-performance" — just restates the abstract. | Name the specific components: vectorized expression evaluator, Arrow-compatible columnar format, plugin-based extensibility for custom types/functions/connectors, adaptive runtime optimizations. What makes it extensible? What makes it high-performance? |
| **System/Evaluation** | Lists components well (type system, vectors, operators). But claims "orders of magnitude speedup" — actual result was ~6-8x. | Be precise with numbers. "Orders of magnitude" means 100x+. Also note production deployment at Meta across multiple systems — stronger than benchmarks alone. |
| **Judgement** | "native-code built, modular, reusable engine that can accelerate query executions" | This restates the paper's claims, not your own analysis. Ask: what could go wrong? (Adoption friction, abstraction leakage across workload types, dependency on Meta's continued investment.) |

**Recurring patterns:**
1. **Restating instead of analyzing** — Judgement sections summarize what the paper says rather than what you think. Force yourself to start with "However..." or "The risk is..."
2. **Vague adjectives without mechanism** — "reusable, extensible, high-performance" are marketing words. Always follow with *how*: "extensible via a plugin registry for custom types and functions" is concrete.

</details>


