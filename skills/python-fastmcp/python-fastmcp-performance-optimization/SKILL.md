---
name: python-fastmcp-performance-optimization
description: Guides teams to optimize and tune FastMCP performance across server, client, and protocol layers — implementing Context-Aware LRU Caching, Capability-Aware Load Balancing, DAG Request Orchestration, HTTP Compression/Connection Pooling, and Sliding-Window Performance Regression Monitors.
---

# Python FastMCP: Performance Optimization & Tuning

This skill helps AI design and implement performance optimization strategies for FastMCP servers and client applications. Because optimizing single components in isolation can increase coordination overhead and break dynamic adaptability, effective performance tuning requires a holistic approach—combining server-side context-aware caching, capability-aware load balancing, DAG request orchestration, protocol compression, and sliding-window regression monitoring.

---

## When to use this skill

Use this skill when you need to:

- reduce end-to-end latency and coordination overhead in multi-server FastMCP architectures,
- implement **Context-Aware LRU Caching** (`ContextAwareCache`) that invalidates or isolates cache entries by query and user context (permissions, role),
- implement **Capability-Aware Load Balancing** (`CapabilityAwareLoadBalancer`) routing requests to the least-loaded server offering a specific capability set,
- orchestrate complex multi-step tool calls as Directed Acyclic Graphs (DAGs) using `asyncio.gather` for maximum parallelism,
- optimize protocol transport using HTTP gzip/brotli payload compression, SSE connection pooling (HTTP keep-alive), and JSON-RPC request batching,
- monitor latency in real time using sliding windows (`PerformanceMonitor`) to automatically flag performance regressions.

---

## Outcome

Produce an optimized FastMCP server/client architecture that:

- caches composite query+context outputs to avoid redundant computation,
- distributes workloads efficiently across heterogeneous FastMCP servers based on capability sets and active load,
- parallelizes independent tool call dependencies using DAG execution,
- tracks mean and P95 latency in sliding windows, triggering alerts when recent latency degrades by >20%.

---

## Instructions for the AI

1. **Implement Server-Side Context-Aware LRU Caching**
   - Cache results using composite keys combining the query string and user context signature (`query::json_context`).
   - Example implementation:
     ```python
     import json
     from typing import Any, Dict, List, Optional

     class ContextAwareCache:
         """LRU Cache keyed on query and user context (permissions, scope)."""

         def __init__(self, max_size: int = 256):
             self.max_size = max_size
             self._store: Dict[str, Any] = {}
             self._access_order: List[str] = []
             self.hits = 0
             self.misses = 0

         def _make_key(self, query: str, context: Dict[str, Any]) -> str:
             ctx_sig = json.dumps(context, sort_keys=True)
             return f"{query}::{ctx_sig}"

         def get(self, query: str, context: Dict[str, Any]) -> Optional[Any]:
             key = self._make_key(query, context)
             if key in self._store:
                 self.hits += 1
                 self._access_order.remove(key)
                 self._access_order.append(key)
                 return self._store[key]
             self.misses += 1
             return None

         def put(self, query: str, context: Dict[str, Any], value: Any) -> None:
             key = self._make_key(query, context)
             if len(self._store) >= self.max_size and key not in self._store:
                 oldest = self._access_order.pop(0)
                 del self._store[oldest]
             self._store[key] = value
             if key not in self._access_order:
                 self._access_order.append(key)

         @property
         def hit_rate(self) -> float:
             total = self.hits + self.misses
             return self.hits / total if total else 0.0
     ```

2. **Implement Capability-Aware Load Balancing**
   - Route incoming tool/resource requests to the server that possesses all required capabilities and has the lowest current active load count.
   - Example load balancer:
     ```python
     class CapabilityAwareLoadBalancer:
         """Routes requests to the least-loaded server matching capability requirements."""

         def __init__(self):
             self.servers: Dict[str, Dict[str, Any]] = {}

         def register_server(self, name: str, capabilities: List[str]) -> None:
             self.servers[name] = {
                 "capabilities": set(capabilities),
                 "current_load": 0,
                 "total_handled": 0,
             }

         def select_server(self, required_caps: List[str]) -> Optional[str]:
             candidates = [
                 (name, info["current_load"])
                 for name, info in self.servers.items()
                 if set(required_caps).issubset(info["capabilities"])
             ]
             if not candidates:
                 return None
             candidates.sort(key=lambda c: c[1])
             chosen = candidates[0][0]
             self.servers[chosen]["current_load"] += 1
             self.servers[chosen]["total_handled"] += 1
             return chosen

         def release(self, server_name: str) -> None:
             if server_name in self.servers:
                 self.servers[server_name]["current_load"] = max(
                     0, self.servers[server_name]["current_load"] - 1
                 )
     ```

3. **Implement Client-Side DAG Request Orchestration**
   - Execute multi-step workflow graphs as Directed Acyclic Graphs (DAGs), identifying tasks whose dependencies are resolved and running them concurrently with `asyncio.gather`.
   - Example orchestrator:
     ```python
     import asyncio
     from typing import Any, Callable, Dict, List

     class RequestOrchestrator:
         """Executes DAGs of tool requests with maximum batch parallelism."""

         async def execute_dag(
             self, dag: Dict[str, List[str]], execute_fn: Callable[[str], Any]
         ) -> Dict[str, Any]:
             completed: Dict[str, Any] = {}
             remaining = dict(dag)
             while remaining:
                 ready = [
                     task
                     for task, deps in remaining.items()
                     if all(d in completed for d in deps)
                 ]
                 if not ready:
                     raise RuntimeError("Cycle detected in task DAG")

                 batch_results = await asyncio.gather(*(execute_fn(t) for t in ready))
                 for task, result in zip(ready, batch_results):
                     completed[task] = result
                     del remaining[task]
             return completed
     ```

4. **Apply Protocol-Level Transport Optimizations**
   - Enable HTTP payload compression (`gzip` or `brotli`) for JSON-RPC messages larger than 1 KB.
   - Maintain persistent SSE HTTP connections (`keep-alive`) to avoid TCP/TLS handshake latency on every tool call.
   - Batch independent JSON-RPC tool calls into a single array payload when executing parallel requests.

5. **Monitor Latency & Detect Regressions**
   - Maintain a sliding window of request latency samples (`window=50`).
   - Flag automated performance regressions when recent window mean latency exceeds prior window mean by >20%.
   - Example monitor:
     ```python
     import statistics

     class PerformanceMonitor:
         """Sliding-window latency recorder and regression detector."""

         def __init__(self, window: int = 50):
             self.window = window
             self._samples: List[float] = []

         def record(self, latency: float) -> None:
             self._samples.append(latency)
             if len(self._samples) > self.window * 2:
                 self._samples = self._samples[-self.window * 2 :]

         def regression_detected(self) -> bool:
             if len(self._samples) < self.window * 2:
                 return False
             old = self._samples[-self.window * 2 : -self.window]
             new = self._samples[-self.window :]
             return statistics.mean(new) > statistics.mean(old) * 1.2
     ```

---

## Decision points and guidance

- **Global LRU Cache vs Context-Aware Cache?** Never use plain query string keys for cached user resources; always include user permissions and tenant scope in the cache key to prevent authorization leaks.
- **When to use DAG Orchestration over Sequential Calls?** Use DAG execution whenever a client needs to invoke 2 or more independent tools (e.g. fetching schema and fetching documents) before passing their joint outputs to a synthesis tool.
- **Connection Reuse Strategy:** Configure `httpx.AsyncClient` or `aiohttp.ClientSession` with persistent connection pools (`max_keepalive_connections=20`) to eliminate TLS renegotiation latency.

---

## Quality criteria

- **Context Isolation:** Cached items include user context signatures in key generation.
- **Parallel Task Execution:** Independent workflow tasks execute concurrently in parallel batches (`asyncio.gather`).
- **Capability-Aware Load Distribution:** Load balancer selects the least-loaded server that satisfies all required capabilities.
- **Automated Regression Detection:** Monitor alerts when sliding-window latency degrades by >20%.

---

## Example prompts

- "Implement a context-aware LRU cache for our FastMCP server queries."
- "Write a DAG request orchestrator for client tool calls using asyncio.gather."
- "Build a capability-aware load balancer for our heterogeneous FastMCP servers."

---

## Related guidance

- python-fastmcp-server-basics
- python-fastmcp-client-integration
- python-fastmcp-evaluation-benchmarking
- python-fastmcp-security-discovery
