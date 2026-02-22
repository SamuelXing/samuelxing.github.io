---
title: 'DB Reading: Lakehouse - A New Generation of Open Platforms that Unify Data Warehousing and Advanced Analytics (2021)'
date: 2026-02-21
tags:
  - databases
  - readings
---

Link: [https://15721.courses.cs.cmu.edu/spring2024/papers/01-modern/armbrust-cidr21.pdf](https://15721.courses.cs.cmu.edu/spring2024/papers/01-modern/armbrust-cidr21.pdf)


![Lakehouse Figure 2](/images/2026-01-31-db-readings-1-10/LakehouseFig2.png)

This paper proposes the Lakehouse, a new data platform architecture that replaces the costly two-tier data lake + warehouse pattern with a single tier built on open storage formats (e.g., Parquet) enriched by a transactional metadata layer (e.g., Delta Lake). By storing data once in an open, directly accessible format while providing warehouse-grade features like ACID transactions, versioning, indexing, and SQL optimization, the Lakehouse eliminates data staleness and redundancy, reduces cost and vendor lock-in, and enables machine learning and analytics tools to work directly over the same data. Evaluated on TPC-DS benchmarks, a Lakehouse system showed performance competitive with leading cloud warehouses, and the paper identifies future work in metadata layer design, new storage formats, caching strategies, and serverless computing.

<details markdown="1">
<summary><b>Detailed Notes</b></summary>


### Why is this paper important?
The Lakehouse is proposed as a replacement for the traditional data warehouse architecture. It is based on three key pillars: open direct-access data formats (e.g., Parquet), first-class support for machine learning and data science, and state-of-the-art performance. This shift addresses major warehouse disadvantages, including data staleness, poor reliability, high total cost of ownership, data lock-in, and limited use-case support.

### What are the key ideas/main contributions?
The paper argues that the current two-tier architecture—where data is first ETLed to a data lake and then ELTed into a warehouse—causes data staleness and creates unnecessary complexity and new failure modes that impact reliability. Data stored in proprietary warehouses is not ideal for advanced analytics or machine learning because these tools cannot efficiently access data via standard warehouse APIs. This dual-tier process is also very costly due to data being saved and transformed in multiple places.

### What system was built/modified, and how was it evaluated?
The proposed architecture combines the key benefits of data lakes (low-cost, directly accessible storage) and data warehouses (ACID transactions, data versioning, auditing, indexing, caching, and query optimization). A Lakehouse enriches raw objects in a data lake with a transactional metadata layer (such as Delta Lake), which implements ACID properties and other management features. The authors evaluated a Lakehouse system using Parquet storage, showing it is competitive with popular cloud data warehouses on TPC-DS performance benchmarks.

### Judgement and follow-up ideas?
This design is very promising because it reduces the overhead of data being transformed and stored in multiple locations. Unlike a traditional DBMS, the data storage format becomes part of the system's public API to allow fast, direct access for diverse workloads. Future work should focus on better designs for the metadata layer, including where transaction logs should be stored and how to support cross-table transactions. There is also a need for better data formats that provide more flexibility for data layout optimizations and indexes while being better suited to modern hardware. Additional research is required for better caching, auxiliary data structures, and data layout strategies, as well as determining how to use serverless computing to minimize query latency.

### Open questions in the paper
*   **Are there other ways to achieve the Lakehouse goals?** One could build a massively parallel serving layer in front of a data warehouse, but this is likely to be more expensive, harder to manage, and less performant than giving workloads direct access to the object store. Standardizing on an open format also ensures direct access to data without being blocked by a vendor.
*   **What are the right storage formats and access APIs?** While Parquet and ORC are the current standard, future storage schemes may include "programmable" decoding logic to offer the system more flexibility.
*   **How does the Lakehouse affect other data management research and trends?** It simplifies data integration and cleaning by allowing tools to run "in place" over all organizational data. It supports the "data mesh" trend by allowing distributed teams to share data directly via object storage without onboarding users to the same compute resources. Finally, it may allow HTAP systems to archive data directly into a Lakehouse for consistent snapshot querying.

</details>
