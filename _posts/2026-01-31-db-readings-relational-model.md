---
title: 'DB Reading: A Relational Model of Data for Large Shared Data Banks (1970)'
date: 2026-02-21
tags:
  - databases
  - readings
---

Link: [https://www.cs.cmu.edu/~15721-f25/papers/A_Relational_Model_of_Data.pdf](https://www.cs.cmu.edu/~15721-f25/papers/A_Relational_Model_of_Data.pdf)

This work is profound. We can see it as the foundation of every modern relational database. Programs should say *what* data they want, not *how* to get it - Data Independencies. In the relational model, relations are value-based, not pointer-based, so the system can reorganize storage freely without breaking applications.

<details markdown="1">
<summary><b>Detailed Notes</b></summary>

### Why is this paper important? (1-2 sentences)
Introduced the relational data model (data represented as mathematical relations/sets of tuples), the concept of a universal data sublanguage (which later inspired SQL), and the principle of data independence.

### What are the key ideas/main contributions? (2-3 sentences)
**Physical Data Independence**: The application should not break when the physical representation of data is changed, e.g., reordering rows on disk, switching from B-tree to hash index, moving a table to a different storage device, or changing the page layout.
**Logical Data Independence**: The application should be resistant to logical schema changes — splitting a table into two, merging tables, adding columns, or restructuring relations. Views can present the old structure on top of a new schema, so existing queries continue to work.

### What system was built/modified, and how was it evaluated? (1-2 sentences)
This is a theoretical paper — no system was built. Codd proposed the relational data model and defined relational algebra and relational calculus as formal query languages that operate on relations.

### Judgement and follow-up ideas or open questions? (1-2 sentences)
This work is profound. We can see it as the foundation of every modern relational database.
Programs should say *what* data they want, not *how* to get it. In the relational model, relations are value-based, not pointer-based, so the system can reorganize storage freely without breaking applications.

</details>
