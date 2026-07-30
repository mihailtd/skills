---
name: python-concurrent-processing
description: >
  Instructs the agent on parallelizing stateless operations using functional map()
  patterns with ProcessPoolExecutor, ThreadPoolExecutor, and multiprocessing.Pool.
  Covers integrating CPU-bound work with asyncio via run_in_executor, data partitioning
  (MapReduce), and managing shared state with locks. Use when parallelizing computations,
  handling CPU-bound work in async apps, or processing large datasets concurrently.
---

# Python Concurrent Processing Guidelines

You are an expert Python developer specializing in functional programming, high-performance concurrency, and asynchronous programming. When asked to parallelize computations, process large datasets, or handle CPU-bound work in an asyncio application, you must adhere to the following rules:

---

## 1. Never Block the Event Loop

Because asyncio uses a single-threaded concurrency model, executing CPU-bound Python code directly inside a coroutine will block the entire event loop and prevent other tasks from running.

*   Do not use standard coroutines for heavy computations.
*   Always offload CPU-bound work to separate processes to bypass the Global Interpreter Lock (GIL) and take full advantage of multi-core CPUs.

## 2. Stateless Concurrency (Embarrassingly Parallel)

*   Do not use complex, stateful objects, locks, shared memory, or queues to pass data if it can be avoided.
*   Construct concurrency around stateless, pure functions that accept argument values and produce discrete results without relying on shared state.

## 3. The Concurrent `map()` Pattern

*   Do not write imperative loops that manually assign work to threads or processes.
*   Use the `map()` method provided by process and thread pools to distribute iterable items to workers.
*   Treat concurrent pools exactly like the built-in `map()` higher-order function: pass the stateless function first, followed by the iterable data source.

## 4. Choosing the Right Executor

*   **CPU-Bound Work:** Use `multiprocessing.Pool` or `concurrent.futures.ProcessPoolExecutor` to bypass the GIL and run computations on multiple cores.
*   **I/O-Bound Work:** Use `concurrent.futures.ThreadPoolExecutor` when processes spend most of their time waiting for network requests, file reading, or database queries.

## 5. Integrating with Asyncio (`run_in_executor`)

To bridge multiprocessing with asyncio, use the `concurrent.futures.ProcessPoolExecutor`.

*   Create the executor using a context manager (`with ProcessPoolExecutor() as pool:`).
*   Use `loop.run_in_executor(pool, func)` to run synchronous CPU-bound functions inside the process pool.
*   Because `run_in_executor` does not accept keyword arguments or multiple arguments directly, always use `functools.partial` to bind arguments to your target function.
*   Use `asyncio.gather` or `asyncio.as_completed` to await the results.

## 6. Partitioning Data (MapReduce)

When processing massive datasets, do not create a separate task for every single item, as the overhead of serializing (pickling) and deserializing data between processes will destroy performance gains.

*   Implement a MapReduce pattern by partitioning the data into smaller, manageable chunks.
*   Submit each chunk to the process pool to be "mapped" (processed).
*   "Reduce" (aggregate) the intermediate results into a final output.

## 7. Shared State and Race Conditions

By default, separate processes do not share memory. If you must maintain shared state:

*   Use `multiprocessing.Value` or `multiprocessing.Array` to allocate shared memory.
*   **Avoid Race Conditions:** Modifying shared data is not atomic. Always synchronize access using a lock. Call `get_lock().acquire()` and `get_lock().release()` (or use a `with` block) around the critical section.
*   **Process Pool Initialization:** Because shared objects cannot be pickled, pass them to worker processes using the `initializer` and `initargs` parameters when creating the `ProcessPoolExecutor`.

## 8. Resource Management

*   Always encapsulate `Pool` or `Executor` objects within a `with` statement (context manager). This ensures worker processes and threads are cleanly terminated and prevents zombie processes.

---

## Code Examples

**1. Using `multiprocessing.Pool` with `imap_unordered`**

```python
import multiprocessing
from collections import Counter
from typing import Iterable

def analyze_file(filename: str) -> Counter[str]:
    # Stateless, pure function that processes a single item
    return Counter()

def process_all_files(file_iter: Iterable[str]) -> Counter[str]:
    combined: Counter[str] = Counter()

    with multiprocessing.Pool() as workers:
        results_iter = workers.imap_unordered(analyze_file, file_iter)
        for result in results_iter:
            combined.update(result)

    return combined
```

**2. Using `concurrent.futures` for I/O or CPU Processing**

```python
from concurrent import futures
from collections import Counter
from typing import Iterable

def process_with_futures(file_iter: Iterable[str], pool_size: int = 4) -> Counter[str]:
    combined: Counter[str] = Counter()

    with futures.ProcessPoolExecutor(max_workers=pool_size) as workers:
        for result in workers.map(analyze_file, file_iter):
            combined.update(result)

    return combined
```

**3. Asyncio Integration with `run_in_executor` and `partial`**

```python
import asyncio
import functools
from concurrent.futures import ProcessPoolExecutor

def cpu_bound_task(data_chunk: list) -> int:
    return sum(data_chunk)

async def main():
    loop = asyncio.get_running_loop()
    data = list(range(100))

    # Partition data into chunks to avoid pickling overhead
    chunks = [data[i:i+10] for i in range(0, len(data), 10)]

    with ProcessPoolExecutor() as pool:
        tasks = [
            loop.run_in_executor(pool, functools.partial(cpu_bound_task, chunk))
            for chunk in chunks
        ]
        results = await asyncio.gather(*tasks)
        print(f"Final reduced result: {sum(results)}")
```

**4. Shared State and Locks in a Process Pool**

```python
import asyncio
import functools
from multiprocessing import Value
from concurrent.futures import ProcessPoolExecutor

shared_counter: Value

def init_worker(counter: Value):
    global shared_counter
    shared_counter = counter

def process_and_count(item: int):
    result = item * item

    # Synchronize access to avoid race conditions
    with shared_counter.get_lock():
        shared_counter.value += 1

    return result

async def main():
    loop = asyncio.get_running_loop()
    counter = Value('i', 0)

    with ProcessPoolExecutor(initializer=init_worker, initargs=(counter,)) as pool:
        tasks = [
            loop.run_in_executor(pool, functools.partial(process_and_count, i))
            for i in range(100)
        ]
        await asyncio.gather(*tasks)

    print(f"Total items processed: {counter.value}")
```
