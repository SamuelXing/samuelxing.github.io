---
title: 'An Empirical Evaluation of Columnar Storage Formats (2023)'
date: 2026-02-22
tags:
  - databases
  - columnar store
  - data format
---

This paper introduces a benchmark framework calibrated to real-world data distributions for a rigorous Parquet vs. ORC comparison. Four design lessons emerge: default dictionary encoding, favoring decode speed over compression ratio, optional block compression, and finer-grained auxiliary structures

Link: [https://15721.courses.cs.cmu.edu/spring2024/papers/02-data1/p148-zeng.pdf](https://15721.courses.cs.cmu.edu/spring2024/papers/02-data1/p148-zeng.pdf)

<details markdown="1">
<summary><b>Detailed Notes</b></summary>

### Why is this paper important? (1-2 sentences)
Prior columnar format benchmarks use synthetic data with unrealistic uniform distributions. This paper introduces a benchmark framework calibrated to real-world data distributions for a rigorous Parquet vs. ORC comparison.

### What are the key ideas/main contributions? (2-3 sentences)
The core contribution is a workload generator that extracts column-level statistical properties (NDV ratio, sortedness, skew) from seven real-world datasets and uses them to produce calibrated synthetic data across five application domains. This enables controlled evaluation — including single-variable sweeps — of file size and scan performance for both formats.

### What system was built/modified, and how was it evaluated? (1-2 sentences)
A configurable benchmark framework: workload generator + evaluation harness using each format's native C++ reader (not Arrow tables) to avoid coupling bias. Evaluated on predefined domain workloads, real datasets, and controlled property sweeps.

### Judgement and follow-up ideas or open questions? (1-2 sentences)
Methodology is sound — real-world-calibrated synthetic data is a clear improvement over uniform benchmarks. Four design lessons emerge: default dictionary encoding, favoring decode speed over compression ratio, optional block compression, and finer-grained auxiliary structures. Open questions: generalizability to cloud-native remote storage (S3/GCS), and the authors' own flags around ML workloads and GPU decoding.


</details>
