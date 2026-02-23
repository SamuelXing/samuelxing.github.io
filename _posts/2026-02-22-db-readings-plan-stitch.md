---
title: 'Plan Stitch: Harnessing the Best of Many Plans (2018)'
date: 2026-02-22
tags:
  - databases
  - query optimization
---

This paper addresses a real gap in production query optimization: existing Reversion-Based Plan Correction (RBPC) only considers whole previously-executed plans, which can overlook efficient subplans hidden across different plans. Plan Stitch fills this gap by opportunistically combining the best subplans from old plans, achieving better results than any single previously-executed plan.


Link: [https://www.cs.cmu.edu/~15721-f25/papers/Plan_Stitching.pdf](https://www.cs.cmu.edu/~15721-f25/papers/Plan_Stitching.pdf)


<details markdown="1">
<summary><b>Detailed Notes</b></summary>

### Why is this paper important? (1-2 sentences)
This paper addresses a real gap in production query optimization: existing Reversion-Based Plan Correction (RBPC) only considers whole previously-executed plans, which can overlook efficient subplans hidden across different plans. Plan Stitch fills this gap by opportunistically combining the best subplans from old plans, achieving better results than any single previously-executed plan.

### What are the key ideas/main contributions? (2-3 sentences)
DBMSs can detect query plan regression by reverting to the cheapest previously-executed plan that is still valid; however, this approach may overlook efficient subplans that exist across different historical plans. Plan Stitch addresses this by automatically and opportunistically stitching together the most efficient subplans from previously-executed plans using an AND-OR graph and dynamic programming, relying on observed execution costs rather than optimizer estimates to keep the approach low-risk.

### What system was built/modified, and how was it evaluated? (1-2 sentences)
Plan Stitch was implemented within Microsoft SQL Server's existing plan correction framework. It was evaluated against RBPC on TPC-DS benchmarks and real customer workloads, measuring plan quality, cost estimation accuracy, coverage, overhead, and stitched plan characteristics.

### Judgement and follow-up ideas or open questions? (1-2 sentences)
The AND-OR graph construction is an elegant and inspiring framework, and the results show significant improvement in plan quality with minimal overhead and low regression risk. However, the approach is fundamentally limited to subplans that have been seen before — a natural follow-up would be whether selectively injecting a small number of optimizer-generated exploratory subplans could expand the search space without sacrificing the safety guarantees.


</details>
