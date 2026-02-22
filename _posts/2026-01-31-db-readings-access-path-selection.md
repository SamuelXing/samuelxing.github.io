---
title: 'DB Reading: Access Path Selection in a Relational Database Management System (1979)'
date: 2026-02-21
permalink: /posts/2026/02/db-readings-access-path-selection/
tags:
  - databases
  - readings
---

Link: [https://www.cs.cmu.edu/~15721-f25/papers/System_R_optimizer.pdf](https://www.cs.cmu.edu/~15721-f25/papers/System_R_optimizer.pdf)

This paper presents the query optimizer for System R, one of the first cost-based optimizers for relational databases. The optimizer uses catalog statistics — such as relation cardinalities, index key counts, and page counts — to estimate the cost of different access paths for both single-table queries and multi-table joins.

<details markdown="1">
<summary><b>Detailed Notes</b></summary>

### Why is this paper important?
This paper is a foundational pillar of database research, introducing the cost-based optimization framework that remains the standard for modern industrial relational systems nearly half a century later. It fundamentally proved that non-procedural languages like SQL could be executed with performance competitive to procedural languages through systematic path selection.

### What are the key ideas/main contributions?
The authors proposed a weighted cost formula (COST=PAGE FETCHES+W×RSI CALLS) that uses catalog statistics and selectivity factors to predict I/O and CPU requirements. A core contribution is the iterative dynamic programming search for join ordering, which prunes the search space while preserving solutions for "interesting orders" that might eliminate the need for future sorting.

### What system was built/modified, and how was it evaluated?
The optimizer was developed for the System R experimental database and was evaluated by comparing its predicted path rankings against actual measured execution resources. The results demonstrated that the optimizer correctly identified the true optimal execution plan in the large majority of cases, even when the absolute predicted cost values were inaccurate.

### Judgement and follow-up ideas?
While established as a landmark classic, later work like the  optimizer  highlighted the need for more granular modeling of CPU costs and buffer utilization to improve accuracy. Current trends are moving toward adaptive query processing that corrects cardinality errors at runtime and Machine Learning models that attempt to learn distributions more effectively than traditional magic numbers


### Cost calculation
The cost calculation in the System R optimizer is a **weighted measure** designed to predict the total resources (I/O and CPU) required to execute a specific query plan. I asked AI to generate the cost formula in Python, here's the result:

```python
"""
System R Optimizer Cost Model
From: Selinger et al., "Access Path Selection in a Relational Database Management System" (1979)
"""

# =============================================================================
# STATISTICS (catalog info for a relation R with index I)
# =============================================================================
# NCARD(R)  - cardinality (number of tuples) of relation R
# TCARD(R)  - number of pages in relation R
# ICARD(I)  - number of distinct keys in index I
# NINDX(I)  - number of pages in index I
# P         - tuples per page = NCARD / TCARD


# =============================================================================
# SELECTIVITY FACTORS
# =============================================================================

def selectivity_equal_value(icard=None):
    """col = value"""
    if icard is not None:
        return 1.0 / icard
    return 1.0 / 10  # default if no index

def selectivity_equal_columns(icard1=None, icard2=None):
    """col1 = col2"""
    if icard1 is not None and icard2 is not None:
        return 1.0 / max(icard1, icard2)
    return 1.0 / 10

def selectivity_range(value, high_key, low_key, has_index=False):
    """col > value (or col < value, symmetric)"""
    if has_index:
        return (high_key - value) / (high_key - low_key)
    return 1.0 / 3  # default

def selectivity_between(val1, val2, high_key, low_key, has_index=False):
    """col BETWEEN val1 AND val2"""
    if has_index:
        return (val2 - val1) / (high_key - low_key)
    return 1.0 / 4  # default

def selectivity_in_list(n_items):
    """col IN (list of n_items values)"""
    return min(n_items / 20.0, 0.5)  # heuristic, bounded

def selectivity_and(f1, f2):
    """pred1 AND pred2 (independence assumption)"""
    return f1 * f2

def selectivity_or(f1, f2):
    """pred1 OR pred2"""
    return f1 + f2 - f1 * f2

def selectivity_not(f):
    """NOT pred"""
    return 1.0 - f


# =============================================================================
# RESULT SIZE (RSI cardinality)
# =============================================================================

def rsi_card(ncard, selectivity_factors):
    """Expected number of tuples after applying predicates."""
    f = 1.0
    for sf in selectivity_factors:
        f *= sf
    return ncard * f


# =============================================================================
# SINGLE-TABLE ACCESS PATH COSTS
#
# COST = page_fetches + W * rsi_calls
#   W = weighting factor (relative CPU cost of an RSI call vs a page I/O)
# =============================================================================

def cost_unique_index_eq(W):
    """Unique index matching an equality predicate: exactly 1 fetch + 1 RSI call."""
    return 1 + W

def cost_clustered_index_matching(nindx, ncard, tcard, match_f, rsicard, W):
    """Clustered index matching one or more boolean factors.

    Args:
        nindx:   pages in index I
        ncard:   cardinality of relation R
        tcard:   pages in relation R
        match_f: product of selectivity factors of MATCHING predicates
        rsicard: expected result cardinality (all predicates applied)
        W:       CPU weighting factor
    """
    return match_f * (nindx + tcard) + W * rsicard

def cost_nonclustered_index_matching(nindx, ncard, match_f, rsicard, W):
    """Non-clustered index matching one or more boolean factors.

    Worst case: each qualifying tuple is on a different page.
    """
    return match_f * (nindx + ncard) + W * rsicard

def cost_clustered_index_no_match(nindx, tcard, rsicard, W):
    """Clustered index, no matching predicates (full index scan)."""
    return nindx + tcard + W * rsicard

def cost_nonclustered_index_no_match(nindx, ncard, rsicard, W):
    """Non-clustered index, no matching predicates (full index scan)."""
    return nindx + ncard + W * rsicard

def cost_segment_scan(tcard, rsicard, W):
    """Sequential segment scan (no index)."""
    return tcard + W * rsicard


# =============================================================================
# JOIN COSTS
# =============================================================================

def cost_nested_loop_join(cost_outer, n_outer, cost_inner):
    """Nested-loop join.

    Args:
        cost_outer: cost of scanning the outer relation
        n_outer:    number of tuples from the outer relation
        cost_inner: cost of one scan of the inner relation
    """
    return cost_outer + n_outer * cost_inner

def cost_sort_merge_join(cost_outer, cost_inner, sort_cost_outer=0, sort_cost_inner=0):
    """Sort-merge join.

    If relations are already sorted on the join column, sort costs are 0.
    Merge itself is linear in the sum of pages.

    Args:
        cost_outer:       cost to retrieve outer in join-column order
        cost_inner:       cost to retrieve inner in join-column order
        sort_cost_outer:  cost to sort outer (0 if already ordered)
        sort_cost_inner:  cost to sort inner (0 if already ordered)
    """
    return sort_cost_outer + sort_cost_inner + cost_outer + cost_inner


# =============================================================================
# EXAMPLE USAGE
# =============================================================================

if __name__ == "__main__":
    # Example: relation EMP with 10,000 rows, 500 pages, index on dept (50 depts)
    NCARD = 10_000
    TCARD = 500
    ICARD_DEPT = 50
    NINDX_DEPT = 10
    W = 0.05  # typical: CPU much cheaper than I/O

    # Query: SELECT * FROM EMP WHERE dept = 'CS'
    sf = selectivity_equal_value(ICARD_DEPT)  # 1/50 = 0.02
    result_size = rsi_card(NCARD, [sf])         # 200 tuples

    print(f"Selectivity factor:  {sf}")
    print(f"Expected result:     {result_size} tuples")

    # Clustered index on dept
    c1 = cost_clustered_index_matching(NINDX_DEPT, NCARD, TCARD, sf, result_size, W)
    print(f"Clustered index:     {c1:.1f}")

    # Non-clustered index on dept
    c2 = cost_nonclustered_index_matching(NINDX_DEPT, NCARD, sf, result_size, W)
    print(f"Non-clustered index: {c2:.1f}")

    # Segment scan
    c3 = cost_segment_scan(TCARD, result_size, W)
    print(f"Segment scan:        {c3:.1f}")

```

</details>
