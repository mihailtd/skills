---
name: python-polars-vs-spark

description: Instructs the agent on recognizing when a dataset has genuinely outgrown Polars (and DuckDB) and needs a distributed engine like Apache Spark — the single-machine-vs-distributed distinction is a difference in kind, not degree; concrete signals worth checking before reaching for Spark; what Spark's data-locality/partitioning/broadcast-join model actually buys once a machine's limits are real; and why the operational cost of a cluster is usually not worth paying for datasets this repository's projects actually deal with.
---

# Polars vs. Apache Spark — When a Dataset Has Actually Outgrown a Single Machine

This repository's projects don't operate at a scale where Spark is warranted —
almost everything that looks like "big data" fits comfortably in Polars or
DuckDB on a single well-resourced machine. This skill exists to calibrate
*when that stops being true*, and to be honest about what a distributed engine
actually costs once it is.

## 1. The Difference Is in Kind, Not Degree

Polars scales **up** — a bigger instance, more cores, more RAM — and DuckDB's
out-of-core engine extends that further by streaming data larger than RAM
from disk without materializing it all at once (see
**database-duckdb-overview**). Both are still fundamentally single-machine,
single-process engines. There's a hard ceiling: no matter how well-optimized,
one machine has a maximum disk size and a maximum I/O throughput.

Spark scales **out** — computation runs across many machines, each operating
on its own slice of the data. This solves a different problem than "make one
machine faster": it's for datasets that don't fit on any single practical
machine at all, or that already live distributed across a cluster with no
feasible way to centralize them first. Reaching for Spark because a job
"feels big" rather than because it has hit this actual ceiling buys none of
its benefit and all of its cost.

## 2. Check These Before Assuming You've Outgrown a Single Machine

- **Does it actually fit?** DuckDB's out-of-core engine and Polars' streaming
  mode handle datasets far larger than RAM on a single modern cloud instance
  — realistically hundreds of GB to low TB, not GB-scale. Confirm the data
  genuinely doesn't fit (or doesn't fit *fast enough*, see below) before
  concluding it needs to.
- **Is the bottleneck compute, not data volume?** If a single machine's cores
  can't finish the job in an acceptable time even though the data fits, that's
  a horizontal-scaling problem `Spark` solves — but so, in many cases, does
  just using a bigger single instance, which is far simpler than standing up
  a cluster. Compare the cost of "rent a bigger machine" against "operate a
  distributed cluster" before assuming the latter is necessary.
- **Is the data already distributed at a scale that can't be centralized?**
  This is Spark's actual home turf: data already produced and stored across
  separate systems or a cluster, at a volume where pulling it all onto one
  machine first isn't feasible. If your data currently lives in one
  PostgreSQL database, one set of Parquet files in object storage, or one
  DuckDB file, it isn't in this situation, regardless of its size.
- **Do you already operate Spark infrastructure?** If a team already runs
  Spark for other pipelines, reusing that infrastructure for a new job has a
  much lower marginal cost than introducing it fresh. Don't let existing
  infrastructure justify reaching for it elsewhere, but don't ignore that
  context either.

## 3. What Spark's Architecture Actually Buys You

Spark (and distributed engines like it) exist to implement **data locality**:
moving a small amount of serialized computation *to* the machines that hold
the data, instead of moving large amounts of data to the computation. This
matters specifically because network transfer is dramatically slower than
local disk, which is itself slower than memory — moving computation avoids
paying network cost for anything except the final, small result.

- **Partitioning** splits data across nodes (by date range, by a hashed key,
  etc.) so each node holds a manageable slice and can process its own slice
  independently.
- **Narrow transformations** (a join where both sides' relevant data already
  live on the same node) need no network transfer at all — the fast case.
- **Wide transformations** (a join or aggregation that requires combining data
  across nodes) require **shuffling** — sending data over the network between
  nodes — which is the expensive case and the main thing distributed-pipeline
  design tries to minimize.
- **Broadcast joins** are the standard optimization when one side of a join is
  much smaller than the other: send the small side to every node holding a
  slice of the large side, instead of shuffling the large side around. This
  only helps when the small side actually fits in each node's memory.

None of this exists in Polars or DuckDB because none of it needs to — a
single-process engine has no network between its "nodes" to shuffle across in
the first place. That absence is a feature, not a gap.

## 4. The Argument Against Reaching for Spark Prematurely Is in Spark's Own Numbers

Network transfer is the slowest resource in a distributed pipeline — orders
of magnitude slower than local disk, which is itself slower than memory.
Every wide transformation (shuffle) in a distributed job pays that network
cost deliberately, as the price of scaling beyond one machine. A single-machine
engine (Polars, DuckDB) never pays it at all, because there's nothing to
shuffle to — everything happens in one process's memory and local disk.

This means the same numbers that justify Spark's architecture for
genuinely-too-large data are the strongest argument for staying on a single
machine as long as the data actually fits: a Polars/DuckDB pipeline is not
just simpler than a Spark job, it structurally avoids the exact cost
(network shuffling) that dominates a distributed pipeline's runtime. "This
might need to scale out eventually" is not, on its own, a reason to pay that
cost today — see **architecture-simplicity** and
**architecture-flexibility-complexity-tradeoffs** for the general version of
this argument against premature generality.

## 5. The Real Cost of Moving to Spark

Beyond the numbers, a Spark migration trades away real simplicity: cluster
provisioning and operation (unless using a fully managed service), a
distributed debugging model instead of a single stack trace, and tuning
concerns with no Polars/DuckDB equivalent — partition sizing, shuffle
minimization, and data skew (one partition key receiving disproportionate
data and becoming a bottleneck all its own). These costs are real regardless
of whether the migration turns out to be worth it, and they should be weighed
explicitly, not treated as a footnote to "Spark can handle more data."

## Decision Heuristic

| Situation | Reach for |
|---|---|
| Data fits on one machine (even if that requires DuckDB's out-of-core engine) | **Polars** (programmatic pipelines) or **DuckDB** (SQL-first, larger-than-memory) — see **database-duckdb-overview** |
| A single machine is too slow, but a bigger single machine would fix it | A bigger machine, not a cluster |
| Data is already distributed across systems at a volume that can't be centralized | **Spark** (or the team's existing distributed infrastructure) |
| "We might need this scale eventually" with no current evidence of it | Stay on Polars/DuckDB — this is the premature-generality trap, not a present requirement |

## Related guidance

- **database-duckdb-overview** — the out-of-core, larger-than-memory engine that extends "fits on one machine" further than Polars alone, and should be checked before assuming Spark is needed.
- **python-polars-vs-pandas** — the separate, already-settled decision of Polars over pandas for in-process DataFrame work.
- **architecture-simplicity** / **architecture-flexibility-complexity-tradeoffs** — the general principle behind section 4: don't pay for flexibility/scale a workload doesn't currently need.
