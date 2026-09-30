# Execution Plans (SQL Server)

*A deep-dive walkthrough of SQL Server execution plans — covering estimated vs. actual plans and why the gap between them is itself diagnostic, how to read a graphical plan (right-to-left, top-to-bottom, cost percentages), the core physical operators (Seeks, Scans, Lookups, and the three join algorithms) and what each one tells you, cardinality estimation and statistics as the engine's actual decision-making input, parameter sniffing as a genuine, common plan-related production issue, plan caching and reuse, and `SET STATISTICS IO/TIME` as the numeric complement to the graphical plan.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Estimated vs. Actual Execution Plans](#1-estimated-vs-actual-execution-plans)
3. [How to Read a Graphical Plan](#2-how-to-read-a-graphical-plan)
4. [Cost Percentages: Relative, Not Absolute](#3-cost-percentages-relative-not-absolute)
5. [Index Seek vs. Index Scan vs. Table Scan](#4-index-seek-vs-index-scan-vs-table-scan)
6. [Key Lookups and RID Lookups](#5-key-lookups-and-rid-lookups)
7. [The Three Join Algorithms](#6-the-three-join-algorithms)
8. [Cardinality Estimation and Statistics](#7-cardinality-estimation-and-statistics)
9. [When Estimated and Actual Rows Diverge: Stale Statistics](#8-when-estimated-and-actual-rows-diverge-stale-statistics)
10. [Parameter Sniffing](#9-parameter-sniffing)
11. [Plan Caching and Reuse](#10-plan-caching-and-reuse)
12. [Missing Index Suggestions](#11-missing-index-suggestions)
13. [SET STATISTICS IO and TIME](#12-set-statistics-io-and-time)
14. [A Worked Example: Diagnosing a Slow Query End to End](#13-a-worked-example-diagnosing-a-slow-query-end-to-end)
15. [Common Pitfalls](#14-common-pitfalls)
16. [Quick Reference Table](#quick-reference-table)
17. [Conclusion](#conclusion)

---

## Introduction

This series' SQL Indexes guide's Section 9 introduces execution plans as the tool that tells you, definitively, whether an index is actually being used — this guide goes considerably deeper into that tool itself: what an execution plan actually represents (the query optimizer's chosen strategy, out of many it considered), the difference between an estimated plan (a prediction) and an actual plan (what genuinely happened), the specific physical operators that show up in a real plan and what each one means for performance, and the statistics-driven cardinality estimation process underneath the optimizer's every decision — including the specific, well-known ways that process can go wrong (stale statistics, parameter sniffing) and produce a plan that looked reasonable to the optimizer but performs badly in practice.

```plaintext
A query goes to the OPTIMIZER → it considers MULTIPLE possible physical
  strategies (which indexes to use, which join algorithm, what order to
  process tables in) → it picks the one with the LOWEST ESTIMATED COST,
  based on STATISTICS about the data (Section 7) → THAT chosen strategy
  is the EXECUTION PLAN — and reading it correctly is how you understand
  (and can influence) what the optimizer actually decided, and why.
```

---

## 1. Estimated vs. Actual Execution Plans

### Estimated plan: what the optimizer PREDICTS will happen, without running the query at all

```sql
-- SQL Server Management Studio: Ctrl+L, or:
SET SHOWPLAN_XML ON;
```

An estimated plan is generated purely from the optimizer's cost-based analysis — it never actually executes the query, which means it's fast to obtain even for a query you're worried might be genuinely slow or resource-intensive to run, but every number in it (row counts, costs) is a *prediction*, based entirely on statistics (Section 7), not what actually happened.

### Actual plan: the SAME plan, annotated with what genuinely occurred during real execution

```sql
SET STATISTICS XML ON; -- or, in SSMS: Ctrl+M ("Include Actual Execution Plan"), then run the query
```

An actual plan requires genuinely running the query — it shows the same chosen strategy (the optimizer decides its plan *before* execution, and doesn't change strategy mid-flight based on what it observes), but now annotated with real, measured numbers: actual rows returned by each operator, actual execution time, actual I/O — critically, this is what lets you directly compare *estimated* rows against *actual* rows at every step, which is Section 8's central diagnostic technique.

### Why you generally want the ACTUAL plan for real diagnostic work

```plaintext
The estimated plan tells you what the optimizer THINKS will happen; the
  actual plan tells you what REALLY happened — and the GAP between the
  two, at any specific operator, is one of the most valuable, concrete
  diagnostic signals available, precisely because it means the
  optimizer's underlying assumptions (Section 7's statistics) were
  wrong for this query, which often explains a genuinely poor performing plan directly.
```

---

## 2. How to Read a Graphical Plan

### Right to left, and top to bottom: the ACTUAL order operations happen in is the reverse of how you might instinctively read it

```plaintext
A graphical execution plan is read RIGHT TO LEFT — the RIGHTMOST
  operators are the FIRST things that happen (reading raw data from a
  table or index); data then flows LEFTWARD, through successive
  operators (joins, filters, sorts), until the LEFTMOST operator
  produces the FINAL result set, which is what gets returned to you.
```

This is worth stating as explicitly as possible, since it's the single most common source of confusion for anyone new to reading these plans — the visual layout genuinely runs opposite to normal left-to-right reading order, and internalizing "rightmost = first, leftmost = last/final result" is the prerequisite for correctly interpreting anything else in the plan.

### Each operator is an icon representing a specific, physical operation

```plaintext
Common icons include: Index Seek, Index Scan, Table Scan (Section 4),
  Key Lookup (Section 5), Nested Loops / Hash Match / Merge Join
  (Section 6), Sort, Filter, Compute Scalar, Aggregate — each icon is a
  DISTINCT, NAMED physical operation the database engine actually
  performs, and arrows between them show the FLOW of rows from one
  operator into the next.
```

### The thickness of the arrows between operators is itself meaningful

```plaintext
Thicker arrows represent MORE ROWS flowing between operators — a
  visually THICK arrow feeding into a later operator is a quick, visual
  signal that a LOT of data is being processed at that point, often
  worth investigating whether that volume was genuinely necessary or a
  sign something upstream (a missing filter, a non-selective predicate)
  is passing more rows forward than it should.
```

---

## 3. Cost Percentages: Relative, Not Absolute

### Every operator shows a percentage — of the TOTAL QUERY's estimated cost, not an absolute, portable number

```plaintext
"Cost: 65%" on a specific operator means "the optimizer estimates this
  operator accounts for 65% of THIS QUERY's total estimated cost" — it
  is a RELATIVE, internal accounting figure, useful for identifying
  which PART of a SPECIFIC query is the most expensive, but NOT a
  meaningful, comparable number across DIFFERENT queries.
```

This is a genuinely important, often-overlooked nuance worth stating precisely — a "cost" of 50 on one query and 50 on an entirely different query do *not* represent the same amount of actual work; these percentages exist purely to help you find the expensive part *within* one query's plan, pointing you toward where to focus your optimization effort, not as a universal, cross-query performance metric.

### Using cost percentage as a triage tool, not a definitive verdict

```plaintext
The operator with the HIGHEST cost percentage is usually, though not
  ALWAYS, the best place to start investigating — a high cost percentage
  on a Table Scan (Section 4) is a strong, common signal worth pursuing;
  but the cost model itself is an ESTIMATE (again, based on statistics,
  Section 7), so treating it as triage GUIDANCE, not gospel, is the
  correct way to use it.
```

---

## 4. Index Seek vs. Index Scan vs. Table Scan

### Index Seek: the fast, targeted B-tree navigation this series' SQL Indexes guide's Sections 2-3 describe

```plaintext
An Index Seek means the optimizer navigated DIRECTLY to the matching
  row(s) using the index's B-tree structure — this is the FAST,
  expected outcome for a well-indexed, selective (per that guide's
  Section 8) query condition, and generally what you WANT to see for a
  query filtering on a specific, narrow value.
```

### Index Scan: reading an ENTIRE index, in order, even though it IS being used

```plaintext
An Index Scan means the optimizer read THROUGH the entire index
  structure (or a large portion of it) rather than navigating directly
  to specific rows — this can still be CHEAPER than a full table scan
  (the index might be narrower/smaller than the full table, or already
  provide the needed sort order for an ORDER BY), but it's NOT the
  targeted, fast lookup an Index Seek represents — worth distinguishing
  these two, since both technically "use an index," but with very different performance profiles.
```

### Table Scan / Clustered Index Scan: reading the ENTIRE table

```plaintext
A Table Scan (on a heap, a table with NO clustered index) or a
  Clustered Index Scan (functionally similar, on a table that HAS one)
  means the ENTIRE table was read, row by row — per this series' SQL
  Indexes guide's Section 3, this is the O(n) outcome that indexing
  exists specifically to avoid, and seeing this on a large table for a
  SELECTIVE query condition is a strong, direct signal that either NO
  useful index exists, or the query optimizer determined (per that
  guide's Section 8) that using an available index wouldn't actually help.
```

This trio directly extends this series' SQL Indexes guide's own Section 9 introduction — worth treating the distinction between "Seek" and "Scan" specifically as the single most important visual signal to look for first in any execution plan you're investigating for a performance problem, since it directly answers "is this query finding its data efficiently, or reading far more than it needs to."

---

## 5. Key Lookups and RID Lookups

### The extra step this series' SQL Indexes guide's Section 7 identifies, made visible in the plan

```plaintext
A Key Lookup (on a table with a clustered index) or RID Lookup (on a
  heap) appearing ALONGSIDE an Index Seek means: the seek found the
  matching row's LOCATION efficiently via a non-clustered index, but the
  query needed ADDITIONAL COLUMNS not included in that index, requiring
  a SEPARATE trip back to the full row for each match.
```

### Why seeing MANY Key Lookups is a specific, actionable signal

```plaintext
A Key Lookup performed ONCE is cheap; a Key Lookup performed for
  THOUSANDS of matching rows (visible in the actual plan's row counts,
  Section 1) is a genuinely expensive, repeated operation — this is
  PRECISELY the signal this series' SQL Indexes guide's Section 7 says
  to look for when deciding whether a covering index (via INCLUDE) would
  be worth adding for this specific, frequently-run query.
```

This is worth treating as a direct, concrete trigger: seeing a Key Lookup operator with a high "actual number of rows" in a frequently-executed query's plan is close to the textbook case for reaching for that guide's covering-index technique — the plan is showing you, precisely, where the extra cost is coming from and what specific optimization would eliminate it.

---

## 6. The Three Join Algorithms

### Nested Loops: for one small input joined against another, using a seek per outer row

```plaintext
For EACH row in the (usually smaller) OUTER input, the engine performs
  a SEEK or scan against the INNER input to find matching rows — this is
  efficient when the outer input is genuinely SMALL and the inner input
  has a USEFUL index to seek against for each outer row; it becomes
  genuinely expensive if the outer input is large, since the inner
  operation repeats ONCE PER outer row.
```

### Hash Match: for joining two LARGE, unsorted inputs

```plaintext
The engine builds an in-memory HASH TABLE from one input (the smaller
  of the two, ideally), then probes it once for each row of the other
  input — this AVOIDS needing an index or existing sort order on either
  side, making it the typical choice for joining two large tables where
  NEITHER Nested Loops (too many outer rows) nor Merge Join (neither
  input is pre-sorted) would be efficient.
```

### Merge Join: for joining two inputs that are ALREADY sorted on the join key

```plaintext
If BOTH inputs are already sorted on the join column (often because an
  index provides that order for free, per this series' SQL Indexes
  guide's Section 2 discussion of linked, sorted leaf nodes), the engine
  can walk BOTH sorted sequences in a single, efficient pass — this is
  often the CHEAPEST option when its specific precondition (both sides
  pre-sorted) genuinely holds, but it's not usable at all otherwise.
```

### Why seeing the "wrong" join algorithm for a given data shape is a real, diagnostic signal

```plaintext
A Nested Loops join between two GENUINELY LARGE, unindexed tables is a
  strong signal something's wrong — the optimizer may have been misled
  by inaccurate row-count ESTIMATES (Section 7-8) into choosing an
  algorithm that's a poor fit for the ACTUAL data volume involved,
  which is precisely why comparing the JOIN algorithm chosen against
  the ACTUAL row counts flowing through it (from an actual plan,
  Section 1) is a genuinely useful diagnostic technique.
```

---

## 7. Cardinality Estimation and Statistics

### The optimizer's entire decision-making process rests on ESTIMATING how many rows each operation will touch

```plaintext
Before choosing Seek vs. Scan, or which JOIN algorithm to use, the
  optimizer needs to ESTIMATE how many rows a given predicate will
  match, and how many rows each intermediate step will produce — this
  estimation process is called CARDINALITY ESTIMATION, and it is
  entirely dependent on STATISTICS the database maintains about the
  actual DISTRIBUTION of values in each column.
```

### What statistics actually contain: a histogram approximating a column's value distribution

```plaintext
SQL Server maintains a HISTOGRAM for indexed (and some non-indexed)
  columns — a compact, approximate summary of how VALUES are
  distributed across the column (which values are common, which are
  rare) — this is what lets the optimizer estimate "WHERE Status =
  'Shipped'" will match roughly 40% of rows, while "WHERE Status =
  'Cancelled'" will match roughly 2%, WITHOUT actually scanning the
  table to count first.
```

This is the concrete, mechanical foundation underneath this series' SQL Indexes guide's Section 8 selectivity discussion — selectivity isn't something the optimizer discovers by inspection at query time; it's read directly from these pre-computed, periodically-updated statistics, which is precisely why Section 8 of *this* guide covers what happens when those statistics fall out of date.

### `UPDATE STATISTICS` and auto-update: keeping the histogram accurate as data changes

```sql
UPDATE STATISTICS Orders; -- manually refresh — SQL Server also does this AUTOMATICALLY,
                             --  typically triggered once a sufficient FRACTION of a table's
                             --  rows have changed since the last update
```

---

## 8. When Estimated and Actual Rows Diverge: Stale Statistics

### The single most valuable comparison an ACTUAL plan enables

```plaintext
Every operator in an actual plan shows BOTH "Estimated Number of Rows"
  AND "Actual Number of Rows" — when these two numbers are CLOSE, the
  optimizer's underlying statistics were accurate for this query, and
  its chosen plan is likely genuinely well-suited to the real data.
  When they DIVERGE SIGNIFICANTLY (an estimate of 100 rows, an actual of
  100,000), the optimizer made its ENTIRE downstream decision-making —
  which join algorithm, whether to seek or scan — based on WRONG
  information, and the resulting plan can be genuinely, badly mismatched
  to the real workload as a direct consequence.
```

### Why this divergence happens: statistics that are OUT OF DATE relative to the table's current, real content

```plaintext
A large batch INSERT, a bulk DELETE, or simply substantial ongoing
  write activity (per this series' SQL Indexes guide's Section 5 write-
  cost discussion) can shift a table's actual data distribution
  meaningfully before the next AUTOMATIC statistics update triggers —
  in the interim, the optimizer is working from a HISTOGRAM that no
  longer accurately reflects reality.
```

This is worth treating as the single highest-value diagnostic technique this whole guide covers — a genuinely poor-performing query, where the *shape* of the chosen plan (a Nested Loops join where a Hash Match would clearly be better, say) looks wrong given the real data volume, is very often explained precisely by finding a large estimated-vs-actual gap at the specific operator where the bad decision originated, and the fix (`UPDATE STATISTICS`, or investigating why auto-update hasn't kept pace) directly follows from that finding.

---

## 9. Parameter Sniffing

### The problem: a cached plan, optimized for ONE parameter value, reused for a very different one

```sql
CREATE PROCEDURE GetOrdersByStatus @Status NVARCHAR(20)
AS
    SELECT * FROM Orders WHERE Status = @Status;
```

```plaintext
The FIRST time this procedure runs — say, with @Status = 'Cancelled'
  (rare, HIGH selectivity, per this series' SQL Indexes guide's Section
  8) — the optimizer generates and CACHES a plan (Section 10) using an
  Index Seek, genuinely optimal FOR THAT VALUE. If the SAME cached plan
  is then REUSED for @Status = 'Pending' (common, LOW selectivity,
  matching a huge fraction of the table), that SAME Seek-based plan
  might now be genuinely, badly suboptimal — a table scan would likely
  have been faster for THIS value, but the cached plan doesn't reconsider.
```

### Why this is a genuine, well-known, recurring production issue — not a rare edge case

```plaintext
Plan caching (Section 10) exists specifically to AVOID the real cost of
  re-optimizing a query on every single execution — "parameter
  sniffing" is the (largely accurate) name for the OPTIMIZER'S normal,
  intended behavior of optimizing based on the FIRST parameter value it
  sees; the PROBLEM only arises when a table's data is genuinely SKEWED
  (some parameter values match far more/fewer rows than others), making
  a single cached plan a poor fit across that whole range of possible values.
```

This is worth understanding as a direct, real-world consequence of Sections 7-8 combined — the optimizer's cardinality-estimation process is inherently tied to *specific* parameter values at the moment a plan is compiled, and reusing that same plan for a parameter value with a very different selectivity profile is precisely where the mismatch originates.

### Mitigations, each with a real trade-off

```sql
-- OPTION query hint: forces the optimizer to estimate as if a SPECIFIC value were used, every time
SELECT * FROM Orders WHERE Status = @Status OPTION (OPTIMIZE FOR (@Status = 'Pending'));

-- RECOMPILE: forces a FRESH plan on EVERY execution — avoids the mismatch entirely,
-- at the cost of paying the FULL optimization cost every single time
SELECT * FROM Orders WHERE Status = @Status OPTION (RECOMPILE);
```

Neither of these is a free win — `OPTIMIZE FOR` trades "optimal for one specific value" for "consistently reasonable across most values," and `RECOMPILE` trades plan-caching's whole performance benefit (Section 10) for guaranteed, per-execution freshness — worth choosing deliberately, based on genuinely observing (via Section 8's estimated-vs-actual comparison, across multiple executions with different real parameter values) that parameter sniffing is a confirmed, real cause of the specific performance problem you're investigating, not applied reflexively.

---

## 10. Plan Caching and Reuse

### Why the optimizer doesn't re-plan every single query execution from scratch

```plaintext
Optimizing a query — considering multiple possible strategies, estimating
  cost for each — is itself genuinely expensive work. SQL Server CACHES
  a compiled plan the first time a given query (or parameterized
  procedure) runs, and REUSES that same cached plan for subsequent,
  identical (or parametrically equivalent) executions, avoiding the
  re-optimization cost entirely for the common case.
```

This is precisely the mechanism Section 9's parameter sniffing issue is a direct, well-known side effect of — plan caching is a genuine, valuable performance optimization in the overwhelming majority of cases (most tables don't have the kind of skewed data distribution that makes reuse a problem), and understanding *why* it exists is what makes Section 9's specific failure mode make sense as a real trade-off, not an arbitrary bug.

### `sys.dm_exec_query_stats` and related DMVs: inspecting what's actually cached

```sql
SELECT qs.execution_count, qs.total_worker_time, st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time DESC; -- the most CUMULATIVELY expensive cached queries
```

Worth knowing these dynamic management views exist for genuinely investigating what's actually cached and how expensive it's been across its real, cumulative execution history — useful for identifying which specific query is worth investigating further with the graphical/actual-plan tools this guide otherwise covers.

---

## 11. Missing Index Suggestions

### SQL Server's own, built-in recommendation, surfaced directly in the graphical plan

```plaintext
A dashed-green "Missing Index" suggestion sometimes appears above a
  plan, recommending a SPECIFIC index (column list and included
  columns) the optimizer estimates would have measurably reduced this
  QUERY's cost, had it existed.
```

### Why this suggestion is worth genuine scrutiny, not automatic adoption

```plaintext
Per this series' SQL Indexes guide's Section 5 and Section 11: EVERY
  suggested index carries a real write-side maintenance cost — the
  Missing Index feature evaluates ONLY this ONE query's read-side
  benefit, with NO visibility into how OFTEN this query actually runs
  relative to writes against the same table, or whether a SIMILAR,
  already-existing index could be adjusted instead of adding an
  entirely new one.
```

This is worth stating as a direct, important caution — treating every Missing Index suggestion as an automatic action item is precisely the "add an index and hope" anti-pattern that guide's Section 11 warns against; the suggestion is a genuinely useful *starting point* for investigation, evaluated against real query frequency and existing index overlap, not a command to execute unconditionally.

---

## 12. SET STATISTICS IO and TIME

### The numeric, non-graphical complement to the visual plan

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
SELECT * FROM Orders WHERE CustomerId = 42;
```

```plaintext
Table 'Orders'. Scan count 1, logical reads 4, physical reads 0, ...
SQL Server Execution Times: CPU time = 15 ms, elapsed time = 22 ms.
```

`STATISTICS IO` reports the actual number of pages read (logical reads specifically — from cache/memory; physical reads specifically indicate genuine disk I/O, a more expensive event) per table touched by the query; `STATISTICS TIME` reports actual CPU and elapsed time — both are genuinely useful as precise, numeric confirmation alongside the graphical plan's visual/relative cost percentages (Section 3), and are often the first thing worth checking when comparing two candidate query rewrites or index strategies against each other empirically.

### Why "logical reads" specifically is often the more stable, comparable metric than elapsed time

```plaintext
Elapsed TIME can vary run to run based on server load, caching state,
  and other concurrent activity — LOGICAL READS (the number of 8KB
  pages the query needed to touch) is a considerably more STABLE,
  repeatable metric for comparing two different approaches to the SAME
  query, since it reflects the actual WORK done, independent of
  momentary server conditions.
```

---

## 13. A Worked Example: Diagnosing a Slow Query End to End

### Putting the whole guide's toolkit together, in the order you'd actually use it

```plaintext
1. Capture the ACTUAL plan (Section 1) for the slow query.
2. Read it right-to-left (Section 2), and check cost percentages
   (Section 3) to find the most expensive operator.
3. Check that operator's TYPE — a Table Scan or Index Scan on a large
   table (Section 4) is an immediate, strong candidate.
4. Compare ESTIMATED vs. ACTUAL rows at that operator (Section 8) — a
   large gap points toward stale statistics as a genuine contributing cause.
5. If a Key Lookup with a HIGH actual row count appears (Section 5),
   that's a direct signal a covering index (per this series' SQL
   Indexes guide's Section 7) would likely help.
6. Check whether the query is a PARAMETERIZED procedure whose
   performance varies dramatically by parameter value (Section 9) —
   if so, parameter sniffing is worth investigating as a contributing cause.
7. Confirm any proposed fix EMPIRICALLY, via STATISTICS IO (Section 12)
   before and after, rather than assuming the fix helped.
```

This is worth treating as the actual, practical workflow this guide's individual sections assemble into — no single technique here is usually sufficient alone; a genuine, real-world slow-query investigation typically moves through several of these steps in sequence, each one narrowing down or confirming a specific, concrete hypothesis about what's actually wrong.

---

## 14. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Relying only on the estimated plan for real diagnostic work | Estimated numbers can diverge significantly from what actually happens, hiding the real problem | Capture and examine the ACTUAL plan, comparing estimated vs. actual rows at each operator (Section 1, Section 8) |
| Reading a graphical plan left to right | The actual order of operations runs opposite to normal reading direction, leading to backwards analysis | Read right to left — the rightmost operators happen first (Section 2) |
| Treating cost percentages as a comparable, absolute metric across different queries | These percentages are relative to one specific query's total estimated cost, not a portable measurement | Use cost percentage only to triage within a single query's plan, not to compare across queries (Section 3) |
| Assuming "uses an index" always means "fast" | An Index Scan reads the whole index, which can be nearly as costly as a table scan for a large index | Distinguish Index Seek (targeted, fast) from Index Scan (reads broadly) explicitly (Section 4) |
| Adopting every "Missing Index" suggestion automatically | Ignores the real write-side maintenance cost and doesn't account for existing, potentially-adjustable indexes | Evaluate missing index suggestions against actual query frequency and existing indexes before adding anything (Section 11) |
| Assuming a stored procedure's performance is stable regardless of which parameter value triggered the cached plan | A plan cached for one, non-representative parameter value can be badly mismatched for other, skewed values | Investigate parameter sniffing specifically when a procedure's performance varies dramatically by input (Section 9) |
| Comparing elapsed time alone to judge whether a query change helped | Elapsed time is noisy, varying with server load and caching state independent of the query itself | Use logical reads (via `STATISTICS IO`) as a more stable, repeatable comparison metric (Section 12) |
| Never investigating whether statistics are stale on a large, frequently-written table | The optimizer's entire decision-making is built on statistics; stale ones produce systematically wrong plan choices | Compare estimated vs. actual rows, and consider `UPDATE STATISTICS`, when a plan's shape looks mismatched to real data volume (Section 7-8) |

---

## Quick Reference Table

| Concept | What It Tells You |
|---|---|
| Estimated plan | The optimizer's prediction, without running the query |
| Actual plan | What genuinely happened, including real row counts per operator |
| Index Seek | Fast, targeted B-tree navigation — the desired outcome for selective queries |
| Index Scan / Table Scan | Broad or full read — expected for low-selectivity queries, a red flag otherwise |
| Key Lookup | An extra retrieval step for columns not covered by the index used |
| Nested Loops / Hash Match / Merge Join | The three join algorithms, each suited to different input sizes/sort states |
| Estimated vs. Actual rows | A large gap signals stale statistics driving a poor plan choice |
| Parameter sniffing | A cached plan, optimal for one parameter value, reused poorly for a skewed one |
| `STATISTICS IO`/`TIME` | Precise, numeric, repeatable measurement complementing the graphical plan |

---

## Conclusion

An execution plan is the query optimizer showing its work — which strategy it chose, out of the many it considered, and (via the actual plan specifically) how well that choice actually held up against real data. Everything this guide covers ultimately traces back to one underlying truth this series' SQL Indexes guide already establishes: the optimizer's decisions are only as good as the statistics feeding its cost model, and the single most valuable diagnostic move available — comparing estimated against actual row counts at each operator — is really just checking whether that underlying model of the data was accurate for this specific query. Stale statistics and parameter sniffing are two distinct, well-understood ways that model can be wrong, and recognizing their specific signatures in a real plan is what turns "this query is slow" into a precise, actionable diagnosis rather than a guess.

Reading a plan fluently — right to left, distinguishing Seeks from Scans, recognizing when a Key Lookup or a mismatched join algorithm is the actual bottleneck — is a genuinely learnable, mechanical skill, not intuition, and it's the direct, practical complement to this series' SQL Indexes guide's more conceptual coverage of selectivity, composite column order, and covering indexes: that guide tells you what a good indexing strategy looks like in principle, and this one tells you how to verify, empirically, whether a specific query is actually benefiting from it.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the estimated-100-rows-actual-2-million-rows discovery that made statistics staleness click far better than any histogram explanation ever could.*
