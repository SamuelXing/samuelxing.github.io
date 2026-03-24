---
title: 'Granularity of Locks and Degrees of Consistency in a Shared Database(1976)'
date: 2026-03-23
published: true
tags:
  - databases
  - transaction
---

This paper established two foundational frameworks that every modern DBMS still uses: (1) multigranularity locking with intention locks (IS, IX, SIX), which allows transactions to lock at the right level without coarse locks blocking fine-grained concurrency, and (2) the four degrees of consistency (0-3), which became the basis for SQL isolation levels (READ UNCOMMITTED through SERIALIZABLE).

Link: [Granularities_of_Locking.pdf](https://www.cs.cmu.edu/~15721-f25/papers/Granularities_of_Locking.pdf)

<details markdown="1">
<summary><b>Detailed Notes</b></summary>

### Why is this paper important? (1-2 sentences)
This paper established two foundational frameworks that every modern DBMS still uses: (1) multigranularity locking with intention locks, which allows transactions to lock at the right level (record, page, table, database) without coarse locks blocking fine-grained concurrency, and (2) the four degrees of consistency (0-3), which became the basis for SQL isolation levels (READ UNCOMMITTED through SERIALIZABLE).

### What are the key ideas/main contributions? (2-3 sentences)
The paper introduces a hierarchical lock tree (database > area > file > record) with **intention locks** (IS, IX, SIX) — before locking a record in S mode, a transaction must acquire IS on every ancestor node, signaling its intent without blocking other transactions at different granularities. It defines a **lock compatibility matrix** that governs which lock types can coexist. Separately, it defines four **degrees of consistency** — degree 0 (no overwriting dirty data), degree 1 (no lost updates = long write locks), degree 2 (no dirty reads = short read locks), degree 3 (repeatable reads = long read locks) — and proves that **two-phase locking (2PL)** guarantees degree 3 (serializability). The key insight is that the locking *protocol* (when you acquire/release) determines the *consistency level* you get.

### What system was built/modified, and how was it evaluated? (1-2 sentences)
This is primarily a theoretical paper with formal proofs. It references System R (IBM) as the practical context, and the multigranularity locking scheme was implemented in System R and its successors (DB2). The evaluation is proof-based rather than experimental.

### Judgement and follow-up ideas or open questions? (1-2 sentences)
The degrees of consistency directly became SQL's isolation levels (degree 1 ≈ READ COMMITTED, degree 3 ≈ SERIALIZABLE), making this one of the most practically influential concurrency control papers. However, the lock-based approach assumes pessimistic concurrency — modern systems increasingly use MVCC (multi-version concurrency control) to achieve similar isolation levels without blocking readers. The paper also doesn't address distributed transactions, where 2PL's requirement to hold locks until commit creates performance problems across network partitions.

### Evaluate my original writeup

| Aspect | Original | How to improve |
|---|---|---|
| **Importance** | "minimize the locks while achieve high concurrency" — captures one half (granularity) but misses the other half entirely (degrees of consistency). | This paper has *two* contributions: (1) multigranularity locking with intention locks, and (2) the formal definition of consistency degrees that became SQL isolation levels. Name both. |
| **Key ideas** | "defined granularities of lock and consistency levels which become a standard" — names the topics without explaining any of them. | Explain the *mechanism*: What are intention locks (IS, IX, SIX) and why do they matter? What are the four degrees? How does 2PL guarantee degree 3? |
| **System** | "no actually system was build" — factually incomplete. | System R implemented this scheme. And the evaluation is proof-based — a formal proof *is* the evaluation for a theory paper. |
| **Judgement** | Empty. | This is the most important section to fill. Ask: what held up? (Intention locks are in every DBMS.) What changed? (MVCC largely replaced lock-based isolation.) |

**Recurring patterns:**
1. **Naming topics without explaining them** — "defined granularities of lock" tells me the paper exists, not that you understood it. Add one sentence of *how*: "intention locks (IS, IX, SIX) allow coarse declarations of intent so fine-grained locks don't block unrelated transactions."
2. **Empty judgement** — The judgement is where *your* thinking goes. Even one sentence helps: "2PL is correct but pessimistic; modern systems prefer MVCC to avoid blocking readers."

</details>
