---
title: '[Morning Reading] Time, Clocks, and the Ordering of Events in a Distributed System'
date: 2026-11-09
permalink: /posts/2026/11/lamport-clock/
published: false
tags:
  - distributed-systems
  - consistency
  - paper-reading
  - UIUC CS 525
---

**Paper:** [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf) — Leslie Lamport, Communications of the ACM 21(7), 1978



### Problem — what problem, and why is it hard? (1-2 sentences)
Distributed systems need to order events, but this is hard because clock drift and there is no shared "now".

### Key insight — the one idea that makes the solution work (1 sentence)
We don't need real time; we only need causal order, plus a fixed process order to break ties.

### How it works — the mechanism, at a level you could re-explain (2-4 sentences)
Each process keeps a counter that increases with every event; on receiving a message, it sets the counter to max(own, Tm) + 1. Ties are broken by process ID, which gives a total order.

### Evidence — how do they show it works? (eval, proof, or argument) (1-2 sentences)
The paper formally proves a bound on how far physical clocks can drift apart in the Appendix, and argues the rest informally.

### Limitations & assumptions — what does it assume, and where does it break? (1-2 sentences)
Timestamps can't detect concurrency, and the mutual exclusion algorithm stops if any process crashes.

### Takeaways — impact, connections to other papers, open questions (1-2 sentences)
This idea is later used in [Dynamo](/posts/2026/05/dynamo/) (vector clocks) and Spanner (physical clocks with TrueTime).

