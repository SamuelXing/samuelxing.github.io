---
title: '[Morning Reading] Distributed Snapshots: Determining Global States of Distributed Systems'
date: 2026-11-11
permalink: /posts/2026/11/distributed-snapshot/
published: true
tags:
  - distributed-systems
  - snapshots
  - paper-reading
  - UIUC CS 525
---

**Paper:** [Distributed Snapshots: Determining Global States of Distributed Systems](https://lamport.azurewebsites.net/pubs/chandy.pdf) — K. Mani Chandy and Leslie Lamport, ACM Transactions on Computer Systems 3(1), 1985

## TL;DR

### Problem
The core problem is how to record a global state in a distributed system. It's hard because there is no shared clock or shared memory, and the snapshot must not stop or change the running computation.

### Key insight
Processes record at different times, but the snapshot is still consistent because the marker separates each channel's messages into those sent before and after the sender recorded its state.

### How it works
After a process records its state, it sends a marker on every outgoing channel, before sending further messages.
When q receives a marker on channel c: if q hasn't recorded yet, it records its own state and records c as empty; otherwise, it records c as the messages received on c after q recorded its state and before the marker.
This only works because channels are FIFO.

### Evidence
Theorem 1 proves that the recorded state S* may never have actually occurred, but it is reachable from the state when the snapshot started, and the state when it ended is reachable from S*. So if a stable property holds in S*, it still holds at the end.

### Limitations
The algorithm assumes channels are FIFO and error-free; if a process crashes during the snapshot, the algorithm gets stuck.
It also only detects stable properties.

### Takeaways
This builds on happens-before from [Lamport clocks](/posts/2026/11/lamport-clock/): a snapshot is consistent if, whenever it includes a message being received, it also includes that message being sent. Flink's checkpoint barriers are the same idea as markers.
