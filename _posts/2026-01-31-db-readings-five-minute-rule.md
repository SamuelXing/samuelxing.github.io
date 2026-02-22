---
title: 'DB Reading: The Five-Minute Rule Ten Years Later (1997)'
date: 2026-02-21
tags:
  - databases
  - readings
---

Link: [https://www.cs.cmu.edu/~natassa/courses/15-721/papers/gray.pdf](https://www.cs.cmu.edu/~natassa/courses/15-721/papers/gray.pdf)

The five-minute rule(for random access) captures the tradeoff between the cost of RAM and the cost of disk I/O. Given the access frequency of a data page, if the page is accessed more often than once every five minutes, it is cheaper to keep it cached in RAM; otherwise, it is cheaper to re-read it from disk on each access.

```python

#variables
Pd = price of one disk drive ($)
A  = random I/Os per second per drive
Pm = price per MB of RAM ($)
p  = page size (e.g., 4 KB)

# Cost of RAM for one page
CostRAM = Pm / PagesPerMB = Pm × p / 10⁶

# Cost of disk I/O capacity to serve 1 access every T seconds:
# One disk provides A accesses/sec. We need 1/T accesses/sec. So we need 1/(T×A) fraction of a disk:
CostDisk = Pd / (T × A)

# At break-even, CostRAM = CostDisk:
Pm × p / 10⁶ = Pd / (T × A)

# Solve for T
T = (Pd / A) / (Pm × p / 10⁶)
      ─────     ───────────────
        │              │
  cost of one     cost of RAM
  I/O per sec     for one page

# Or simply
BreakEven T = CostPerIOPerSec / CostPerPageOfRAM
```
