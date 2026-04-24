---
name: asyncio-aiohttp-concurrent-web-requests
description: Instructs the agent on making non-blocking HTTP requests with aiohttp, and managing concurrent execution using asyncio.gather, asyncio.as_completed, and asyncio.wait, including timeout management.
---

# Asyncio Concurrent Web Requests Guidelines

You are an expert Python developer specializing in asynchronous programming. When asked to implement web requests, scrape websites, or interact with external REST APIs, you must adhere to the following rules:

## 1. Non-Blocking I/O with `aiohttp`

Never use the standard `requests` library in an asynchronous context, as it uses blocking sockets and will halt the entire `asyncio` event loop [2-4].

- Always use `aiohttp` for non-blocking HTTP requests [4, 5].
- Use `aiohttp.ClientSession()` as an asynchronous context manager (`async with`) to take advantage of connection pooling [6]. Do not create a new session for every request.
- **Timeouts:** Configure timeouts using the `aiohttp.ClientTimeout` data structure, which can be applied to the entire session or individual requests [7].

## 2. Running Requests Concurrently

When making multiple web requests, run them concurrently to improve application throughput [8, 9]. Do not await requests sequentially in a loop [10, 11].

- **`asyncio.gather`:** Use this when you need all requests to finish before processing results [9, 12]. Always consider setting `return_exceptions=True` so that a single failed request does not crash the entire batch; instead, exceptions will be returned alongside successful results [13, 14].
- **`asyncio.as_completed`:** Use this when you want to process results immediately as they come in, rather than waiting for the slowest request to finish [12, 15]. It returns an iterator of futures that yield results as soon as they complete [15, 16].

## 3. Finer-Grained Control with `asyncio.wait`

Use `asyncio.wait` when you need granular control over the execution state of your concurrent tasks, such as canceling running requests if one fails [17, 18].

- Ensure you wrap all coroutines in tasks (using `asyncio.create_task()`) before passing them to `wait` so you can accurately track and compare them [19-22].
- `wait` returns two sets: `done` and `pending` [18, 23].
- **Fail Fast:** Pass `return_when=asyncio.FIRST_EXCEPTION` to immediately return the `done` and `pending` sets if a request fails, allowing you to cancel the `pending` tasks [24, 25].
- **Group Timeouts:** You can pass a `timeout` argument to `wait`. Unlike `wait_for`, it does not raise a `TimeoutError`. Instead, it returns the tasks that finished within the time limit in the `done` set, and leaves the running ones in the `pending` set. _You must manually loop through the `pending` set and call `.cancel()` on them if you want to stop their execution_ [20, 26].

---

## Code Examples

Below are best-practice examples demonstrating how to properly use `aiohttp` alongside `asyncio` concurrency APIs.

**1. Basic Setup & `asyncio.gather` (Wait for all, handle exceptions)**

```python
import asyncio
import aiohttp

async def fetch_status(session: aiohttp.ClientSession, url: str) -> int:
    # Set a specific timeout of 5 seconds for the request
    timeout = aiohttp.ClientTimeout(total=5.0)
    async with session.get(url, timeout=timeout) as response:
        return response.status

async def fetch_all_gather():
    urls = ['https://example.com' for _ in range(10)]

    # Use a single session for all requests to utilize connection pooling
    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(fetch_status(session, url)) for url in urls]

        # return_exceptions=True prevents one failed request from cancelling the others
        results = await asyncio.gather(*tasks, return_exceptions=True)

        for result in results:
            if isinstance(result, Exception):
                print(f"Request failed: {result}")
            else:
                print(f"Status: {result}")
2. Immediate Processing with asyncio.as_completed
async def fetch_as_completed():
    urls = ['https://example.com' for _ in range(10)]

    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(fetch_status(session, url)) for url in urls]

        # Process each result exactly when it finishes
        for finished_task in asyncio.as_completed(tasks, timeout=10.0):
            try:
                result = await finished_task
                print(f"Finished early with status: {result}")
            except asyncio.TimeoutError:
                print("A request timed out.")
3. Advanced Control and Cancellation with asyncio.wait
async def fetch_with_wait():
    urls = ['https://example.com', 'https://bad-url-example.com', 'https://example.com']

    async with aiohttp.ClientSession() as session:
        # Crucial: wrap in tasks to accurately track pending vs done
        tasks = [asyncio.create_task(fetch_status(session, url)) for url in urls]

        # FIRST_EXCEPTION allows us to fail fast if one request errors out
        done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_EXCEPTION, timeout=5.0)

        # Manually cancel any remaining tasks if we failed fast or timed out
        for pending_task in pending:
            pending_task.cancel()

        for done_task in done:
            if done_task.exception() is None:
                print(f"Success: {done_task.result()}")
            else:
                print(f"Failed with: {done_task.exception()}")
```
