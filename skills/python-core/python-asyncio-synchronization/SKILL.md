---
name: python-asyncio-synchronization

description: Instructs the agent on resolving single-threaded concurrency bugs in asyncio, protecting critical sections with Locks, limiting concurrent tasks with Semaphores, and notifying worker tasks with Events and Conditions.
---

# Asyncio Synchronization Guidelines

You are an expert Python developer specializing in asynchronous programming. When asked to resolve single-threaded concurrency bugs, protect shared states, limit task concurrency, or coordinate worker tasks, you must adhere to the following rules:

## 1. Preventing Single-Threaded Race Conditions with `Lock`

Even though asyncio uses a single-threaded concurrency model, race conditions can still occur if a shared mutable state is modified across an `await` suspension point (because other coroutines can run while the original is paused).

- Use `asyncio.Lock` to protect these critical sections.
- Always acquire the lock using an asynchronous context manager (`async with lock:`).
- Do not declare the `Lock` as a global variable outside of a coroutine before the event loop is created, as this can tie it to a different event loop and cause crashes.

## 2. Limiting Concurrency with Semaphores

When you need to control access to a finite resource (e.g., limiting the number of concurrent external API requests to avoid rate limits or overloading), do not use a standard `Lock`.

- Use `asyncio.Semaphore(value)` to allow a specific number of concurrent tasks.
- Use `asyncio.BoundedSemaphore(value)` if you want to ensure that `release()` is never called more times than `acquire()`. It will raise a `ValueError` if the permit count exceeds the initial value.
- Always manage semaphore acquisition via the `async with` context manager.

## 3. Notifying Tasks with `Event`

When you need one or more tasks to remain idle until a specific external action or initialization occurs, use `asyncio.Event`.

- Use `await event.wait()` to block the task until the event is triggered.
- Use `event.set()` in the producer/notifier task to wake up all waiting tasks.
- Use `event.clear()` to reset the internal flag to `False` so future calls to `wait()` will block again.

## 4. Complex Coordination with `Condition`

If you need exclusive access to a shared resource _and_ need to wait for a specific state to be true, use `asyncio.Condition` (which combines a Lock and an Event).

- First, acquire the condition's lock using `async with condition:`.
- Once acquired, use `await condition.wait()` or `await condition.wait_for(predicate)` to release the lock and block until notified.
- To wake up waiting tasks, the notifier must also acquire the lock (`async with condition:`) and call `condition.notify_all()`.

---

## Code Examples

Below are best-practice examples demonstrating how to use asyncio synchronization primitives.

**1. Protecting a Critical Section with `Lock`**

```python
import asyncio

user_names_to_sockets = {'John': 'Socket1', 'Eric': 'Socket2'}

async def user_disconnect(username: str, user_lock: asyncio.Lock):
    # Protect the dictionary modification with a lock
    async with user_lock:
        socket = user_names_to_sockets.pop(username, None)
        if socket:
            print(f'Closed socket for {username}')

async def message_all_users(user_lock: asyncio.Lock):
    # Ensure no users are removed while we are sending messages
    async with user_lock:
        for user, socket in user_names_to_sockets.items():
            await asyncio.sleep(0.1)  # Simulated I/O
            print(f'Sent message to {user}')
2. Limiting Concurrent API Requests with Semaphore
import asyncio
import aiohttp

async def get_url(url: str, session: aiohttp.ClientSession, semaphore: asyncio.Semaphore):
    # Ensures only a set number of requests happen concurrently
    async with semaphore:
        async with session.get(url) as response:
            return response.status

async def main():
    # Limit to exactly 10 concurrent requests
    api_semaphore = asyncio.BoundedSemaphore(10)

    async with aiohttp.ClientSession() as session:
        tasks = [
            get_url('https://example.com', session, api_semaphore)
            for _ in range(100)
        ]
        results = await asyncio.gather(*tasks)
3. Waking up Worker Tasks with Event
import asyncio

async def worker(event: asyncio.Event, worker_id: int):
    print(f'Worker {worker_id} waiting for event...')
    # Block until the event is set
    await event.wait()
    print(f'Worker {worker_id} triggered and doing work!')

async def main():
    start_event = asyncio.Event()

    # Start workers (they will pause at event.wait())
    workers = [asyncio.create_task(worker(start_event, i)) for i in range(3)]

    await asyncio.sleep(2) # Simulate some initialization time
    print('Initialization complete, triggering workers...')

    # Wake up all waiting workers
    start_event.set()
    await asyncio.gather(*workers)
```
