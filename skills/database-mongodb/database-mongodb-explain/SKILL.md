---
name: database-mongodb-explain
description: Guides the agent on MongoDB explain(), query planning, execution statistics, plan interpretation, and query tuning with explain output.
---

# MongoDB explain()

You are an expert on MongoDB query explain plans. When asked about query tuning or optimizer behavior, explain how `explain()` works, how to read `winningPlan`, how to compare alternate plans, and how to use execution statistics to make indexing and query-shape decisions.

## 1. What `explain()` does

- `explain()` reveals how MongoDB chooses to execute a query or aggregation.
- The optimizer evaluates candidate plans and selects the one with the lowest estimated work.
- `explain()` is essential for performance tuning because it exposes indexes, stage order, and execution metrics.

## 2. Two common forms

### Collection-level explain

```js
var explainCsr = db.customers.explain().find({
  FirstName: 'RUTH',
  LastName: 'MARTINEZ',
  Phone: 496523103
}, { Address: 1, dob: 1 }).sort({ dob: 1 });
var explainDoc = explainCsr.next();
printjson(explainDoc.queryPlanner.winningPlan);
```

### Cursor-level explain

```js
var explainDoc = db.customers.find({
  FirstName: 'RUTH',
  LastName: 'MARTINEZ',
  Phone: 496523103
}, { Address: 1, dob: 1 }).sort({ dob: 1 }).explain('executionStats');
```

- The first form is preferred; the second syntax is deprecated.
- Always call `.next()` on the explain cursor if using the collection-level form.

## 3. Explain verbosity levels

- `queryPlanner` (default): shows the chosen plan and rejected alternatives, but does not execute the query.
- `executionStats`: executes the query and returns runtime metrics such as `executionTimeMillis`, `totalKeysExamined`, and `totalDocsExamined`.
- `allPlansExecution`: executes the query and reports metrics for all candidate plans, useful when diagnosing plan selection issues.

### Example

```js
var explainDoc = db.customers.explain('executionStats').find({
  FirstName: 'RUTH',
  LastName: 'MARTINEZ',
  Phone: 496523103
}).sort({ dob: 1 });
printjson(explainDoc.executionStats);
```

## 4. Reading `winningPlan`

- Start from the innermost `inputStage` and read outward.
- Common stages:
  - `COLLSCAN`: full collection scan.
  - `IXSCAN`: index scan.
  - `FETCH`: fetch documents after index filtering.
  - `SORT`: blocking sort in memory.
  - `PROJECTION_SIMPLE` / `PROJECTION_COVERED`: projection of requested fields.
  - `SORT_KEY_GENERATOR`: build sort keys before sorting.
- In the example above, the winning plan may be:
  - `IXSCAN` on `Phone_1`
  - `FETCH` to apply `FirstName` and `LastName` filters
  - `SORT_KEY_GENERATOR` to collect `dob` values
  - `SORT` to order by `dob`
  - `PROJECTION_SIMPLE` to return only `Address` and `dob`

## 5. Understanding alternate plans

- `queryPlanner.rejectedPlans` contains rejected candidates.
- Rejected plans show what MongoDB considered but did not use.
- Use rejected plans to understand whether alternate indexes were evaluated.
- Example:

```js
mongoTuning.quickExplain(explainDoc.queryPlanner.rejectedPlans[1]);
```

- A rejected plan may show `AND_SORTED` or merged index usage.
- Rejections often happen when the optimizer estimates higher cost than the winning plan.

## 6. Execution statistics

- `executionStats` reveals real runtime behavior.
- Important fields:
  - `executionTimeMillis`
  - `totalKeysExamined`
  - `totalDocsExamined`
  - `nReturned`
- `executionStages` mirrors `winningPlan` but includes metrics per stage.

### Example

```js
printjson(explainDoc.executionStats);
```

### Key diagnostics

- `totalKeysExamined` is zero on a `COLLSCAN`.
- `totalDocsExamined` should be close to `nReturned` for efficient plans.
- A large gap between `totalDocsExamined` and `nReturned` often indicates post-index filtering.
- If `executionTimeMillis` is high and a `SORT` stage appears, adding a supporting index may remove the sort.

## 7. Use explain() to tune queries

### Step 1: examine the current plan

- Run `explain('executionStats')` on the real query.
- Identify whether MongoDB uses `COLLSCAN` or an `IXSCAN`.
- Check whether the plan requires a blocking `SORT`.

### Step 2: compare with available indexes

- If only `COLLSCAN` is used, check `db.collection.getIndexes()`.
- If `SORT` appears, determine whether a compound index can cover both filter and sort fields.

### Step 3: create or adjust indexes

- Create indexes that match equality filters first, then sort fields, then range filters.
- Example:

```js
db.customers.createIndex({
  Country: 1,
  'views.title': 1,
  City: 1,
  LastName: 1
}, { name: 'ExplainExample' });
```

### Step 4: re-run explain

- Confirm the winning plan changes from `COLLSCAN` to `IXSCAN`.
- Confirm the number of documents examined drops.
- Confirm `SORT` disappears if the index supports sort order.

## 8. Example query tuning flow

### Original plan

```js
var explainDoc = db.customers.explain('executionStats').find({
  Country: 'United Kingdom',
  'views.title': 'CONQUERER NUTS'
}, { City: 1, LastName: 1, phone: 1 }).sort({ City: 1, LastName: 1 });
```

- If this yields `COLLSCAN`, the query reads every document and performs an in-memory sort.
- If the query scans 411,121 documents, add a targeted index.

### Improved plan

```js
var explainDoc = db.customers.explain('executionStats').find({
  Country: 'United Kingdom',
  'views.title': 'CONQUERER NUTS'
}, { City: 1, LastName: 1, phone: 1 }).sort({ City: 1, LastName: 1 });
```

- After creating the supporting index, the winning plan should be:
  - `IXSCAN` on the new index
  - `FETCH`
  - `PROJECTION_SIMPLE`
- Execution stats should shrink from 411,121 docs examined to a few hundred.

## 9. Practical explain best practices

- Use `explain('executionStats')` on representative production queries, not ad hoc queries only.
- Avoid `executionStats` on large bulk writes or extremely expensive queries unless you need them.
- Use `allPlansExecution` only when investigating why the optimizer chose a nonobvious plan.
- Compare `queryPlanner.winningPlan` with `executionStats.executionStages` to verify runtime behavior.
- Use concise utility helpers like `mongoTuning.quickExplain()` or shell scripts to flatten nested plans.

## 10. Tips and tricks

- When you see `SORT_KEY_GENERATOR`, MongoDB is preparing sort values from document fields.
- Use `IXSCAN` plus `FETCH` when the index covers the query predicate but not all returned fields.
- A `PROJECTION_COVERED` plan means the query can be satisfied entirely from the index.
- If `rejectedPlans` includes multiple index scans, MongoDB is considering index intersection or OR-style plans.
- If a plan uses `COLLSCAN`, think about adding an index on the most selective field in the predicate.
- If your query uses multiple equality clauses, the optimizer may choose the single best index or merge indexes; the chosen plan depends on cost estimates.
- For `sort()` without an index, the plan shows a `SORT` stage. Adding a supporting index can eliminate it.
- If query shape changes frequently, explain the actual query shape you expect in production, not a simplified example.

## 11. Answering style

- Compare alternative query strategies in plain terms: collection scan, index scan, fetch, sort.
- Emphasize the rule: `explain()` shows what MongoDB plans to do, not what it should do.
- Use real shell-style examples and show how to interpret nested `inputStage` structure.
- Mention `executionStats` cost on server resources as a caution.
- Suggest indexes only when explain proves they would change the plan.
