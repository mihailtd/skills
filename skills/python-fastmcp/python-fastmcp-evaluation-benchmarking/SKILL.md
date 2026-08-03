---
name: python-fastmcp-evaluation-benchmarking
description: Guides teams to build Evaluation Frameworks and Automated Benchmarks for FastMCP systems — evaluating capability discovery, P95 latency distributions, throughput under concurrent load (asyncio.gather), reliability/error recovery, LLM-as-a-Judge output quality, and persistent SQLite benchmark logging.
---

# Python FastMCP: Evaluation & Benchmarking

This skill helps AI design and implement automated evaluation frameworks and benchmarking suites for FastMCP servers and client systems. Because MCP systems adapt dynamically to changing server capabilities and exhibit non-deterministic LLM behaviors, evaluation requires multi-dimensional assessment—measuring capability discovery, P95 latency distributions, concurrent throughput, error recovery rates, and task completion metrics persisted into an evaluation database.

---

## When to use this skill

Use this skill when you need to:

- build automated test and benchmark suites for FastMCP servers and multi-agent systems,
- evaluate **Capability Discovery** under minimal, rich, dynamic, and unavailable server environments,
- benchmark system performance across multiple load levels (mean, median, P95 latency, requests per second),
- evaluate **Reliability & Fault Recovery** when servers time out, throw authentication errors, or drop connection pipes,
- evaluate **Task Completion & User Experience** using invariant assertions and LLM-as-a-Judge evaluators for non-deterministic responses,
- log persistent evaluation sessions to SQLite/PostgreSQL for CI/CD quality gates and performance drift monitoring.

---

## Outcome

Produce an automated FastMCP evaluation framework that:

- runs multi-dimensional test suites (Capability Discovery, Performance, Reliability, UX),
- calculates statistical distributions (mean, median, standard deviation, P95 latency, RPS throughput),
- logs structured test results and session summaries into persistent database tables (`evaluation_results`, `evaluation_sessions`),
- enforces pass/fail quality gates based on predefined SLA criteria (e.g. mean latency < 1.0s, pass rate > 95%).

---

## Instructions for the AI

1. **Setup Persistent Evaluation Database**
   - Store test outcomes, latency metrics, error logs, and session summaries in SQLite or PostgreSQL for trend tracking over time.
   - Example schema & framework setup:
     ```python
     import asyncio
     import json
     import sqlite3
     import statistics
     import time
     from dataclasses import dataclass
     from datetime import datetime
     from typing import Any, Dict, List, Optional

     @dataclass
     class EvaluationResult:
         test_name: str
         success: bool
         duration: float
         metrics: Dict[str, Any]
         errors: List[str]
         timestamp: str

     class MCPEvaluationFramework:
         """Automated evaluation framework for FastMCP servers and clients."""

         def __init__(self, db_path: str = "mcp_evaluation.db"):
             self.db_path = db_path
             self._setup_database()

         def _setup_database(self) -> None:
             conn = sqlite3.connect(self.db_path)
             cursor = conn.cursor()
             cursor.execute("""
                 CREATE TABLE IF NOT EXISTS evaluation_results (
                     id INTEGER PRIMARY KEY AUTOINCREMENT,
                     test_name TEXT,
                     session_id TEXT,
                     success BOOLEAN,
                     duration REAL,
                     metrics TEXT,
                     errors TEXT,
                     timestamp TEXT
                 )
             """)
             cursor.execute("""
                 CREATE TABLE IF NOT EXISTS evaluation_sessions (
                     session_id TEXT PRIMARY KEY,
                     start_time TEXT,
                     end_time TEXT,
                     total_tests INTEGER,
                     passed_tests INTEGER,
                     failed_tests INTEGER,
                     summary_metrics TEXT
                 )
             """)
             conn.commit()
             conn.close()
     ```

2. **Dimension 1: Evaluate Capability Discovery**
   - Verify that client sessions correctly discover tools (`list_tools`), handle dynamic tool additions (`list_changed`), and handle missing or failing servers gracefully.
   - Target metrics: `servers_discovered`, `total_capabilities`, `dynamic_capabilities_added`, `unavailable_servers_handled`.

3. **Dimension 2: Evaluate Performance & Concurrency**
   - Measure response times across sequential requests and compute statistical distributions (mean, median, standard deviation, P95 percentile).
   - Test throughput under concurrent load using `asyncio.gather` across multiple concurrency levels (1, 5, 10, 20 workers).
   - Target metrics: `response_time_mean`, `response_time_p95`, `throughput_rps`, `scalability_results`.

4. **Dimension 3: Evaluate Reliability & Error Recovery**
   - Simulate fault scenarios: timeouts, malformed requests, auth failures, missing resources, and server overloads.
   - Verify that the system falls back gracefully to secondary tools or returns structured error codes (`-32602`, `-32000`) without crashing the application process.
   - Target metrics: `error_recovery_success_rate`, `graceful_degradation_handled`.

5. **Dimension 4: Evaluate Non-Deterministic UX & LLM Outputs**
   - Test non-deterministic LLM outputs using **Invariant Assertions** (verifying response schema compliance, SLA latency budget, zero PII leakage) and **LLM-as-a-Judge** scoring for helpfulness and accuracy.
   - Target metrics: `task_completion_rate`, `response_accuracy_score`.

6. **Evaluate Consistency via $pass^k$ Metric vs $pass@k$**
   - Measure tool call reliability using the **$pass^k$ metric** (distinct from $pass@k$), evaluating whether an agent succeeds consistently across all $k$ repeated trials rather than succeeding occasionally in a single trial.
   - Measure exact-match correctness for tool names and JSON Schema argument validation conforming to Berkeley Function Calling Leaderboard (BFCL) and ToolBench standards.

7. **Scalability Concurrency Ramps & Multi-Step End-to-End Scenarios**
   - Ramp concurrent users across levels (1, 5, 10, 25, 50 concurrent requests) to evaluate peak throughput (RPS) and minimum success rate.
   - Run realistic multi-step scenarios (`doc_search`, `multimodal`, `simple_query`, `collab_analysis`) to measure complex workflow completion rates under coordination overhead.

8. **Weighted Dimension Scoring & Comparative Analysis**
   - Calculate weighted average scores across evaluation dimensions (`capability_discovery`, `coordination`, `scalability`, `end_to_end`) to produce an overall system score (0.0 to 1.0 scale).
   - Perform comparative benchmarking across alternative server configurations, deployments, or model backends to track performance evolution and detect regressions over time.

9. **Execute Session & Enforce Quality Gates**
   - Run all evaluation suites, record results into SQLite/PostgreSQL, and compute session pass rate.
   - Raise quality gate errors in CI/CD if overall pass rate drops below SLA thresholds (e.g. < 95%).

---

## Decision points and guidance

- **Exact Match vs Invariant Assertions?** Never use exact string matching for LLM outputs. Assert invariant conditions (schema validity, keyword presence, citation inclusion, latency bounds) or use LLM-as-a-Judge scoring.
- **SQLite vs PostgreSQL for Benchmarking?** Use SQLite (`WAL` mode) for local CI/CD pipelines and single-runner test environments. Migrate to PostgreSQL when multiple worker threads execute concurrent evaluation tasks.
- **How to handle non-deterministic performance outliers?** Calculate 95th percentile (P95) latency alongside mean/median to identify network jitter and server cold-start outliers.

---

## Quality criteria

- **Multi-Dimensional Coverage:** Framework evaluates capability discovery, performance distributions, error recovery, and UX task completion.
- **Statistical Rigor:** Performance metrics capture mean, median, standard deviation, and P95 latency.
- **Persistent Telemetry:** Test results and session summaries are recorded in persistent database tables for drift tracking.
- **CI/CD Integration:** Evaluation sessions output clear pass/fail status and summary metrics.

---

## Example prompts

- "Build an automated evaluation framework for our FastMCP server that measures P95 latency and throughput under load."
- "Write a capability discovery benchmark testing how our MCP client handles unavailable servers and dynamic tool updates."
- "Create an evaluation database logger that records FastMCP test session metrics into SQLite."

---

## Related guidance

- python-fastmcp-manual-testing
- python-fastmcp-security-discovery
- python-fastmcp-client-integration
