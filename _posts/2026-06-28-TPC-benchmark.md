---
title: 'Database Benchmarks: The Categories, the TPC Family, and What the Numbers Actually Mean'
date: 2026-06-28
permalink: /posts/2026/06/db-benchmarks/
tags:
  - databases
  - benchmarks
  - tpc
  - oltp
  - olap
  - ai-teach-me-something
---

A working tour of how the database industry measures itself: the taxonomy of benchmarks, the 40-year arc that produced TPC, what's actually inside TPC-C / TPC-H / TPC-DS, who's winning in 2026, and how to read a benchmark number without getting played.

## The setup: every benchmark is three choices

Every database benchmark is really three choices — **schema shape**, **query mix**, and **what you measure**. Group them by workload character, not by vendor.

| Category | Canonical benchmarks | What's measured | The shape |
|---|---|---|---|
| **OLTP** | TPC-C, TPC-E, TATP, sysbench `oltp_*` | tpmC, tpsE, NOPM, TPS @ p99 | Many short ACID txns, hot rows, write-heavy, latency-sensitive |
| **OLAP** | TPC-H, TPC-DS, SSB | QphH, QphDS, per-query latency, throughput @ N streams | Few long read-only queries, full scans, joins, aggregations |
| **HTAP** | CH-benCHmark, HTAPBench, HyBench | tpmC *and* QphH on one DB | TPC-C txns and TPC-H-style analytics on the *same* tables |
| **Streaming** | NEXMark, Yahoo Streaming Bench | events/sec, end-to-end p99 | Windowed queries over an event stream |
| **KV / NoSQL** | YCSB A–F | ops/sec @ latency | Single-table, single-record ops with parameterized R/W mix |
| **Time-series** | TSBS (DevOps, IoT) | metrics/sec ingest, query latency | High-cardinality writes, range/aggregation reads |
| **Vector** | ANN-Benchmarks, BigANN | QPS at fixed recall@k | Approximate k-NN over high-dim embeddings |
| **Graph** | LDBC SNB Interactive/BI, Graphalytics | response time, throughput | k-hop neighborhood traversals; BFS/PageRank/WCC |
| **JSON/Document** | JSONBench, mgbench | scan + filter throughput | Nested document filters, projection |
| **Micro** | TATP, sysbench, SSB | isolated op latency | A single primitive (point read, range, group-by) under load |

### The YCSB workloads, memorized

YCSB is the lingua franca of KV benchmarking ([Cooper et al., SoCC 2010](https://dl.acm.org/doi/10.1145/1807128.1807152)). The six core workloads:

- **A** — 50/50 read/update, Zipfian (session store)
- **B** — 95/5 read/update, Zipfian (photo tagging)
- **C** — 100% read, Zipfian (profile cache)
- **D** — 95/5 read/insert, *latest* distribution (status feeds)
- **E** — 95/5 short range scan/insert (threaded conversations)
- **F** — read-modify-write (user profile updates)

**Plain-English rule.** If a vendor quotes "1M ops/sec on YCSB" without naming the workload letter and the distribution, ignore it.

---

## How we got here: 40 years of trying to measure databases honestly

**Pre-1985 — ad-hoc and adversarial.** Vendors published "our system runs query X in Y seconds" with no reproducibility. The first systematic break was the **Wisconsin Benchmark** (Bitton, DeWitt, Turbyfill, 1983): single synthetic table, 32 SQL statements, deterministic data generator. Oracle's numbers came out so badly that [Larry Ellison reportedly tried to get DeWitt fired](https://danluu.com/anon-benchmark/) — which is why every Oracle license still contains a **"DeWitt clause"** forbidding you from publishing benchmarks without permission. Other commercial DBs followed.

**1985 — "Anon et al."** Jim Gray and ~20 co-authors publish "[A Measure of Transaction Processing Power](https://jimgray.azurewebsites.net/papers/ameasureoftransactionprocessingpower.pdf)" in *Datamation* under a pseudonym because their employers wouldn't approve it. It defines **DebitCredit / TP1**, the TPS metric, and the `$/tps` price-performance idea. This is the direct ancestor of TPC-A.

**1988 — TPC founded.** Eight vendors form the Transaction Processing Performance Council explicitly to end "benchmarketing." The lineage from there:

| Year | Benchmark | Role | Status |
|---|---|---|---|
| 1989 | TPC-A | DebitCredit with network-attached terminals | Retired |
| 1990 | TPC-B | TPC-A minus terminals, pure DB stress | Retired |
| 1992 | **TPC-C** | Warehouse order-entry OLTP | Still canonical |
| 1995 | TPC-D | First decision-support, 17 queries | Retired |
| 1999 | **TPC-H** / TPC-R | Ad-hoc vs. reporting split; H survived | H still active |
| 2006 | **TPC-DS** | Snowflake schema, 99 queries, realistic warehouse | Active |
| 2007 | TPC-E | Brokerage OLTP, more realistic than C | Active but rarely cited |
| 2014+ | TPCx-HS, BB, AI | "Express" — kit-based, less audit overhead | Active |

The decision-support family didn't replace each other cleanly. TPC-H stayed popular because it's small (22 queries) and easy to run; TPC-DS is closer to reality but punishing to implement. **SSB** ([O'Neil et al., 2009](https://www.cs.umb.edu/~poneil/StarSchemaB.pdf)) is the columnar-friendly simplification of TPC-H.

**The open-source response.** YCSB (2010) for KV. **OLTPBench** (Difallah, Pavlo, Curino, Cudré-Mauroux, VLDB 2013), now [BenchBase](https://github.com/cmu-db/benchbase) at CMU — one JDBC harness for ~19 workloads (TPC-C, TPC-H, YCSB, TATP, AuctionMark, Wikipedia, CH-benCHmark) under one config. CH-benCHmark (2011) glued TPC-C and TPC-H onto the same tables and effectively defined HTAP benchmarking. LDBC formalized graph benchmarks; [ANN-Benchmarks](https://ann-benchmarks.com/) became the de facto vector-search yardstick — QPS-at-recall curves, not single numbers.

### What makes a benchmark "good"

Gray's [*Benchmark Handbook*, 1993](https://jimgray.azurewebsites.net/benchmarkhandbook/chapter1.pdf) names four properties — and they're in tension:

- **Relevance** — resembles real usage
- **Portability** — runs on multiple systems unchanged
- **Scalability** — same workload, parameterized data size
- **Simplicity** — results are interpretable

Add two from later practice: **auditability** (TPC-C requires a full disclosure report and an auditor signature; TPCx-* deliberately relaxes this) and **longevity** — Gray's informal test: if a benchmark survives a decade of hardware turnover, the abstraction was right. TPC-C has survived 30+ years through disk → SSD → NVMe → persistent memory, which is its own argument.

---

## TPC-C — the OLTP standard (1992)

**Scenario.** A wholesale supplier with `W` warehouses, each with 10 districts and 30k customers per warehouse, ordering against a 100k-item catalog.

**Schema.** Nine tables: `WAREHOUSE`, `DISTRICT`, `CUSTOMER`, `STOCK`, `ITEM`, `ORDERS`, `ORDER_LINE`, `HISTORY`, `NEW_ORDER`. Everything except `ITEM` scales linearly with warehouse count.

**Workload.** Five transactions with a fixed mix:

| Transaction | Mix | What it does |
|---|---|---|
| New-Order | 45% | Insert order header + 5–15 lines; for each line read `ITEM`, lock-and-decrement `STOCK.s_quantity`, increment `DISTRICT.d_next_o_id` |
| Payment | 43% | Update `WAREHOUSE.w_ytd`, `DISTRICT.d_ytd`, `CUSTOMER.c_balance`; insert into `HISTORY`. 15% remote-warehouse |
| Order-Status | 4% | Read-only: lookup customer's latest order + lines. Hits a secondary index on `(c_w_id, c_d_id, c_last)` |
| Delivery | 4% | Batch: for each of 10 districts, pop oldest `NEW_ORDER`, mark delivered, sum into `CUSTOMER.c_balance` |
| Stock-Level | 4% | Read-only count of items below threshold for last 20 orders. Only txn allowed READ COMMITTED |

**Metric.** `tpmC` — *only* New-Order transactions per minute count. The other four must still meet timing constraints but don't add to the score.

**Constraints.** ACID compliance is audited (the spec includes explicit isolation and durability tests). 90th-percentile response time must be ≤ 5 s for the first four transactions, ≤ 20 s for Stock-Level. The scaling rule is the hard one: you cannot raise `tpmC` by buying faster hardware against a fixed dataset — the spec caps `tpmC` at roughly **12.86 × W**, so a million-tpmC result requires ~78,000 warehouses, and the database grows accordingly.

**What it measures.** Lock-manager throughput, latch contention on hot pages, log-writer bandwidth, B-tree insert behavior under skew, and the optimizer's ability to pick index-only paths. The famous hotspot is `DISTRICT.d_next_o_id` — every New-Order serializes on its district row, which makes the benchmark a torture test for row-locking, MVCC garbage, and 2PL deadlock avoidance.

**Where it fails.** At any scale that fits the working set in RAM (most modern hardware), storage I/O largely disappears and TPC-C becomes a CPU + concurrency-control microbenchmark. The transaction mix is frozen in 1992 retail. And practically nobody runs the audited version anymore — what you actually see is HammerDB's **TPROC-C**, a fair-use derivative that reports **NOPM** (New Orders Per Minute) because the TPC trademark forbids using `tpmC` for unaudited runs.

## TPC-H — the decision-support standard (1999)

**Scenario.** A parts supplier, eight tables: `LINEITEM`, `ORDERS`, `PART`, `PARTSUPP`, `CUSTOMER`, `SUPPLIER`, `NATION`, `REGION`. `LINEITEM` dominates — ~6×10⁹ rows at SF1000, ~6×10¹¹ at SF100000.

**Workload.** 22 read-only queries (Q1–Q22) plus two refresh functions: **RF1** inserts ~0.1% new orders/lineitems, **RF2** deletes the same fraction of old ones. Defined scale factors: 1, 10, 30, 100, 300, 1000, 3000, 10000, 30000, 100000 GB.

**Metric.** `QphH@Size` = `sqrt(Power × Throughput)`.

- **Power** is the geometric mean of single-stream Q1–Q22 + RF1 + RF2 times, scaled.
- **Throughput** runs `S` concurrent query streams (S grows with SF) plus a serial refresh stream.

The geometric mean is deliberate: it prevents a vendor from winning by being merely excellent at long queries while regressing on the cheap ones.

**What it measures well.** Sequential scan throughput on `LINEITEM` (Q1 is the canonical "scan + group-by"), hash-join cost and build-side sizing (Q5 joins 6 tables), aggregation, sort and TOP-N (Q3, Q10), correlated subqueries (Q17, Q20, Q22), and the optimizer's ability to choose join order across the eight-table schema.

**Where it fails.** The 22 queries are public and stable since 1999. Every vendor has hand-tuned plans, and many ship hints, layouts, or precomputed aggregates specifically for them. The spec forbids materialized views in audited runs, but unaudited runs (the vast majority you see) routinely cheat. There are no window functions, no recursive CTEs, no LATERAL joins, no late-binding semantics — queries are the shape a 1999 query writer would produce.

## TPC-DS — the modern decision-support benchmark (2006)

Nambiar and Poess's "[The Making of TPC-DS](https://www.vldb.org/conf/2006/p1049-othayoth.pdf)" (VLDB 2006) explicitly set out to fix TPC-H's "too small, too simple, too memorizable" problems.

**Scenario.** A retailer selling through three channels: store, catalog, web.

**Schema.** 24 tables — seven fact tables (`store_sales`, `store_returns`, `catalog_sales`, `catalog_returns`, `web_sales`, `web_returns`, `inventory`) and 17 dimensions. A snowflake-with-star-shortcuts shape that defeats trivial denormalization.

**Workload.** **99 query templates** generated from SQL-99 + OLAP extensions, categorized as reporting, ad-hoc, iterative OLAP, and data-mining. Heavy use of window functions, `ROLLUP`/`CUBE`/`GROUPING SETS`, complex CTEs, correlated subqueries. Plus a **Data Maintenance** phase that performs full ETL — inserts, deletes, and Type-1/Type-2 slowly-changing-dimension updates against the live warehouse. Scale factors: 1, 10, 100, 1000, 3000, 10000, 30000, 100000 GB.

**Metric.** `QphDS@Size`, a composite over load time, power test, throughput test, and data maintenance.

**What it measures.** Optimizer robustness under unfamiliar query shapes (this is the point — many queries deliberately resemble plans no human would write), partition pruning across multi-level date partitions, window-function execution, spill behavior on large hash aggregates, and the cost model's ability to handle skewed dimension joins.

**Where it fails.** Auditing is so expensive that fewer than a dozen full audited results exist in 20 years. Almost every vendor (Snowflake, Databricks, ClickHouse, DuckDB, Trino, Vertica) publishes "TPC-DS-style" numbers running a subset of the 99 queries against a precomputed columnar layout, no concurrent streams, and no data maintenance — which is to TPC-DS what a single bench press is to a powerlifting meet.

## The rest: TPC-E, TPC-DI, TPCx-AI, TPC-Energy

- **TPC-E (2007)** — brokerage OLTP, 33 tables, 12 transaction types. ~9:1 read/write ratio (vs TPC-C's ~3:2), realistic data distributions (US Census names, NYSE/NASDAQ tickers). Technically the better OLTP benchmark; lost the network effect because only Microsoft consistently published results.
- **TPC-DI (2014)** — end-to-end ETL: bulk load, transformations, Type-1/Type-2 SCD handling. The only audited ETL benchmark; niche.
- **TPCx-AI (v1 2021, v2 2024)** — retail ML pipeline benchmark with **10 use cases** (classification, clustering, forecasting, anomaly detection, NLP, recommendation). Measures the whole pipeline — ingestion, preprocessing, training, serving — not just model FLOPs.
- **TPC-Energy (2010)** — a modifier, not a standalone benchmark. Adds `Watts/KtpmC`, `Watts/KQphH`, etc. as optional secondary metrics on existing TPC results.

---

## What the production numbers actually look like in 2026

Audited TPC submissions have collapsed since ~2013–2015. A full audit costs $50k+ in auditor fees and consumes months of engineering work for a 200+ page Full Disclosure Report. What's left splits in three buckets.

### Top audited TPC-C

| Year | System | tpmC | $/tpmC | Notes |
|---|---|---|---|---|
| 2010 | Oracle SPARC SuperCluster (T3, 27 servers) | 30,249,688 | $1.01 | The classic Oracle "30M" record |
| 2013 | Oracle SPARC T5-8 (single system) | 8,552,523 | $0.55 | Last big single-system audit |
| 2010 | IBM Power 780 (DB2 9.7) | 10,366,254 | $1.38 | DB2 high-water mark |
| 2020 | Alibaba OceanBase (1,557 servers) | 707,351,007 | ~$0.61 | Shared-nothing scale-out, first non-traditional vendor at the top |
| **2025** | **Alibaba Cloud PolarDB (2,340 nodes)** | **2,055,076,649** | **~$0.11** | Current world record, audited Jan 2025 |

Western vendors (Oracle, IBM, Microsoft, NEC, HPE NonStop) haven't submitted an audited TPC-C result in roughly a decade. The action has moved to Chinese hyperscalers and to unaudited HammerDB runs.

### Top audited TPC-H

| SF | Year | Vendor | QphH |
|---|---|---|---|
| 1 TB | 2021 | Exasol on AMD EPYC | 6,116,949 |
| 3 TB | 2019 | Actian Vector (single-node HPE) | 2,140,307 |
| 10 TB | 2019 | Exasol on Dell R6415 / EPYC 7551P | 8,667,578 |
| 100 TB | — | (no recent audited submission) | — |

The last truly headline-grabbing TPC-H audit was Exasol's 2021 SF1000 result. Cloud DWs abandoned the benchmark.

### TPC-DS — mostly unaudited marketing

Only two fully audited TPC-DS@100TB results exist: Alibaba (14.86M QphDS, 2018) and **Databricks (32.94M QphDS, Nov 2021)** at 2.2x Alibaba and ~10% lower cost. After that, every major cloud DW switched to unaudited "TPC-DS-like" derived benchmarks.

### OLTP beyond TPC-C — HammerDB TPROC-C

| Engine | NOPM | Configuration |
|---|---|---|
| Postgres 18 | ~237K | vu=40, wh=4000, large server |
| MySQL 8 | ~230K | same |
| SQL Server / Oracle on premium HW | >1M | bare-metal enterprise |

Postgres 17/18 and MySQL 8 are within 5% of each other on HammerDB. Commercial (SQL Server, Oracle) leads by 3–5x on top-end hardware but is rarely audited.

### KV / YCSB

ScyllaDB's [Jan 2026 results](https://www.scylladb.com/2026/01/13/scaling-performance-comparison-vs-cassandra/) show tablet-based Scylla running **~7.2x faster than Cassandra vNodes** during scale-out and sustaining **~3.5x higher throughput**. Independent third-party tests usually land at 2–4x — still large, smaller than vendor headlines. FoundationDB and Aerospike own the "deterministic latency at high TPS" niche. Redis is untouchable on single-shard latency.

### The community shootouts

- **[ClickBench](https://benchmark.clickhouse.com/)** is the de-facto OLAP shootout. ClickHouse, StarRocks, and Umbra trade #1 on flat-table aggregation; Snowflake, Druid, and Pinot are 5–50x slower per cold query.
- **[Mark Litwintschik's 1.1B taxi rides](https://tech.marksblogg.com/benchmarks.html)** is the community reference for single-table aggregation. Recent ClickHouse runs finish all 4 queries in **<1s on a single Core i9-14900K**; DuckDB hit the same in single-digit seconds on a laptop.
- **GigaOM** and **Fivetran** run derived TPC-H / TPC-DS cloud-DW comparisons annually. Most show Snowflake / Redshift / BigQuery / Databricks within ~2x on geomean runtime — nobody runs away with it.

### The benchmark wars (Nov 2021 – 2022)

Databricks announced their TPC-DS@100TB world record in Nov 2021 and cited a Barcelona Supercomputing Center re-run claiming Databricks SQL was **2.7x faster and 12x better $/perf vs Snowflake**. Snowflake [fired back](https://www.snowflake.com/en/blog/industry-benchmarks-and-competing-with-integrity/) accusing Databricks of running against a stale, pre-baked Snowflake demo dataset with caches and clustering disabled. Databricks [countered](https://www.databricks.com/blog/2021/11/15/snowflake-claims-similar-price-performance-to-databricks-but-not-so-fast.html) that Snowflake had quietly rewritten the demo dataset two days after the original test, and that an apples-to-apples reload from the official TPC-DS generator still took Snowflake 1.9x longer.

[Yellowbrick's writeup](https://yellowbrick.com/blog/research/and-the-winner-of-the-cloud-data-warehouse-benchmark-war-isnobody) is the community-consensus take: **neither side is fully honest, treat all vendor self-benchmarks as marketing.**

---

## What the number actually means: four traps

**1. Throughput without a latency distribution is a lie.** Dean & Barroso's "[The Tail at Scale](https://cacm.acm.org/research/the-tail-at-scale/)" (CACM 2013) makes the math unavoidable: if a request fans out to 100 backends each with p99 = 10 ms, end-to-end p99 is **~140 ms**, and **63% of user requests hit at least one slow backend**. Quote p50/p99/p99.9 or don't bother. [Tell-Tale Tail Latencies](https://arxiv.org/pdf/2107.11607) (Kersten et al., 2021) shows most published DB benchmarks still get this wrong via under-running, coordinated omission, or GC pauses hidden in the client.

**2. Peak ≠ sustained.** A TPC-C run that hits 2M tpmC for 30 seconds and collapses when the LSM compacts or the WAL fsync queue fills is not a 2M tpmC system. TPC specs require a sustained measurement interval (2 hours for TPC-C) precisely because of this.

**3. Price-performance exists for a reason.** Vendors learned in the late 1990s to win TPC-C with $30M+ custom configurations nobody would buy. `$/tpmC` and `$/kQphH` reset the playing field by making "throw hardware at it" visible in the score.

**4. A 2015 number does not transfer to 2025.** NVMe changed the I/O floor by ~100x. Columnar engines and SIMD changed TPC-H by another 10x. Cloud object storage rewrote the cost model entirely. Treat any cross-decade comparison as a fairy tale.

### The plain-English rule

> If a benchmark result doesn't tell you the **workload, scale factor, concurrency, latency distribution, and cost**, it tells you nothing.

---

## Misconceptions worth retiring

**"TPC-C is irrelevant — it's from 1992."** The workload is from 1992. The systems pressure it exerts (lock-manager throughput, log-writer bandwidth, B-tree concurrency, hot-row contention) hasn't aged a day. Every new OLTP engine still gets a HammerDB TPROC-C run before launch.

**"TPC-H is a good optimizer benchmark."** It was. Now it's a memorization benchmark. Every commercial vendor's TPC-H plan is hand-tuned. Real optimizer robustness lives in TPC-DS (99 queries no one memorizes) or in [JOB — the Join Order Benchmark](https://github.com/gregrahn/join-order-benchmark) (real-world IMDb data, real query shapes).

**"More tpmC = better DB."** Only at comparable price, comparable hardware, comparable transaction mix, comparable durability settings. The 2.05B tpmC PolarDB result used 2,340 nodes. Your single-box workload is not in the same conversation.

**"Vendor X's blog says they're 5x faster than vendor Y."** Read it as marketing. Both Databricks and Snowflake have been caught running each other's systems with non-default settings. The only TPC numbers with full price/availability disclosure and an independent auditor are at [tpc.org/results](https://www.tpc.org/results).

**"Audited benchmarks are dead, so just use HammerDB."** HammerDB is fine for in-house regression testing. It is not an apples-to-apples cross-vendor comparison — different schemas warm differently, different optimizers handle the same SQL differently, and there's no auditor checking your ACID test.

**"Run TPC-DS to know if a warehouse is good."** Run *your queries* on *your data* at *your scale*. TPC-DS exists to find the parts of an engine that break under analytical pressure. It does not predict your workload.

---

## When to reach for which benchmark (decision framework)

```
What are you trying to learn?
├─ "Does this OLTP engine scale on my hardware?"
│    → HammerDB TPROC-C at a warehouse count that exceeds RAM.
│      Watch NOPM and 99th-percentile latency, not just throughput.
│
├─ "Does this analytical engine handle real queries?"
│    → TPC-DS subset on Parquet/columnar at SF1000.
│      Don't trust geomean alone — look at the worst 10 queries.
│
├─ "Optimizer regression detection."
│    → JOB (Join Order Benchmark). Real IMDb data, real query shapes,
│      no vendor has memorized it.
│
├─ "KV throughput sanity check."
│    → YCSB workloads A, B, F. Pick distribution that matches your access pattern.
│
├─ "Buying decision between two cloud DWs."
│    → Your own workload on a free trial. Vendor benchmarks won't transfer.
│
└─ "Storage-engine deep dive (LSM vs B-tree, compaction behavior)."
     → YCSB at sustained load > working set, observed for hours, not minutes.
```

### The five things to write down before publishing any benchmark

1. **Workload** — exact name and version (TPC-C v5.11, YCSB workload A, etc.)
2. **Scale factor** — data size; same workload at SF1 and SF10000 tests different code paths
3. **Concurrency** — number of streams or virtual users
4. **Latency distribution** — at minimum p50/p99; preferably p50/p99/p99.9
5. **Cost** — hardware spec and price, or cloud SKU and hourly cost

Anything missing turns the result into marketing.

---

## The one-line distillations

> **The categories.** OLTP / OLAP / HTAP / KV / streaming / time-series / vector / graph. Pick the one whose *shape* matches your workload; ignore numbers from other categories.
>
> **The arc.** Wisconsin (1983) proved you could measure DBs reproducibly. "Anon et al." (1985) added TPS and price-performance. TPC (1988) added auditing. The audited regime mostly collapsed after 2013; HammerDB and unaudited "TPC-DS-style" runs replaced it.
>
> **TPC-C.** OLTP torture test. Five-transaction mix, `tpmC` counts only New-Order. Measures lock manager, log writer, and hot-row contention. Capped at ~12.86 × W, so high scores require massive databases.
>
> **TPC-H.** 22 fixed queries. Measures scan and hash join. Vendors have memorized it; treat results as upper bounds.
>
> **TPC-DS.** 99 queries the optimizer has not seen before. Closest thing to a real warehouse workload, almost never audited.
>
> **The current top.** PolarDB 2.05B tpmC (Jan 2025, audited). Databricks 32.94M QphDS@100TB (Nov 2021, audited). Everyone else is unaudited.
>
> **Reading a result.** Workload + scale + concurrency + latency distribution + cost. Missing any one, it's marketing.

The short version: **benchmarks are useful as workloads, not as scoreboards.** Run the spec on your hardware, your data, your latency budget. The published numbers tell you what's *possible* on the configurations vendors picked; they don't tell you what's *yours*.

---

## Further reading

- Anon et al., [*A Measure of Transaction Processing Power*](https://jimgray.azurewebsites.net/papers/ameasureoftransactionprocessingpower.pdf) (1985) — the seed paper
- Jim Gray (ed.), [*The Benchmark Handbook*](https://jimgray.azurewebsites.net/benchmarkhandbook/chapter1.pdf) (1993) — chapter 1 is the canon
- Nambiar & Poess, [*The Making of TPC-DS*](https://www.vldb.org/conf/2006/p1049-othayoth.pdf) (VLDB 2006)
- Cooper et al., [*Benchmarking Cloud Serving Systems with YCSB*](https://dl.acm.org/doi/10.1145/1807128.1807152) (SoCC 2010)
- Difallah et al., [*OLTP-Bench*](https://dl.acm.org/doi/10.14778/2732240.2732246) (VLDB 2013) — now [BenchBase](https://github.com/cmu-db/benchbase)
- Dean & Barroso, [*The Tail at Scale*](https://cacm.acm.org/research/the-tail-at-scale/) (CACM 2013) — the latency essay you can't skip
- Kersten, Leis, Neumann, [*Tell-Tale Tail Latencies*](https://arxiv.org/pdf/2107.11607) (2021) — how DB benchmarks get latency wrong
- TPC specs: [TPC-C](https://www.tpc.org/tpc_documents_current_versions/pdf/tpc-c_v5.11.0.pdf) · [TPC-H](https://tpc.org/TPC_Documents_Current_Versions/pdf/tpc-h_v3.0.0.pdf) · [TPC-DS](https://www.tpc.org/tpc_documents_current_versions/pdf/tpc-ds_v3.2.0.pdf) · [TPC-E](https://www.tpc.org/tpc_documents_current_versions/pdf/tpc-e_v1.14.0.pdf)
- [TPC official results](https://www.tpc.org/results) — the only audited numbers
- Dan Luu, [*That time Oracle tried to get a professor fired for benchmarking their DB*](https://danluu.com/anon-benchmark/) — the DeWitt-clause backstory
- [ClickBench](https://benchmark.clickhouse.com/) · [JOB](https://github.com/gregrahn/join-order-benchmark) · [Mark Litwintschik's benchmarks](https://tech.marksblogg.com/benchmarks.html)
