---
title: '[Morning Reading] Time, Clocks, and the Ordering of Events in a Distributed System'
date: 2026-11-09
permalink: /posts/2026/11/lamport-clock/
published: true
tags:
  - distributed-systems
  - consistency
  - paper-reading
  - UIUC CS 525
---

**Paper:** [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf) — Leslie Lamport, Communications of the ACM 21(7), 1978



### Problem
Distributed systems need to order events, but this is hard because clock drift and there is no shared "now".

### Key insight
We don't need real time; we only need causal order, plus a fixed process order to break ties.

### How it works
Each process keeps a counter that increases with every event; on receiving a message, it sets the counter to max(own, Tm) + 1. Ties are broken by process ID, which gives a total order.

### Evidence
The paper formally proves a bound on how far physical clocks can drift apart in the Appendix, and argues the rest informally.

### Limitations & assumptions
Timestamps can't detect concurrency, and the mutual exclusion algorithm stops if any process crashes.

### Takeaways
This idea is later used in Dynamo (vector clocks) and Spanner (physical clocks with TrueTime).

