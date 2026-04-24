---
name: python-api-pagination-streaming
description: Instructs the agent on designing robust API pagination avoiding offset pitfalls, using time-based cursors, and implementing memory-safe asynchronous data streaming using database cursors (via asyncpg or SQLAlchemy).
---

# Python API Pagination & Data Streaming Guidelines

You are an expert Python developer specializing in API design and high-performance database interactions. When asked to implement pagination, filtering, or retrieval of large datasets, you must adhere to the following rules:

## 1. Avoid Strict Offset Pagination Pitfalls
Basic offset pagination (using `page` and `size` parameters) is problematic when data changes frequently between client requests [1]. 
*   If a client fetches Page 1, and a new item is added to the database before they fetch Page 2, an item that was previously on the first page gets pushed to the second page [2]. 
*   This shifting data bug causes clients to retrieve and display the exact same element twice or miss elements entirely [2].

## 2. Implement Time-Based Cursor Pagination
To avoid the shifting data problem, design the API to use time-based cursors instead of strict page numbers.
*   Always sort paginated values by a stable attribute, such as their creation date, ensuring new resources are consistently added to the end [3].
*   For resources that frequently generate new elements, implement an `updated_since` query parameter [3]. This allows the client to request only the resources that are new since their last access timestamp, speeding up queries and guaranteeing no duplicate returns [3].
*   Return a structured JSON response containing `next` and `previous` fields with hyperlinks to the adjacent cursor states, and a `result` array [4].

## 3. Memory-Safe Asynchronous Data Streaming
When dealing with queries that return millions of rows, pulling all data into memory at once will severely tax system RAM and network bandwidth [5]. 
*   Instead of materializing entire result sets, use asynchronous database cursors to stream results only as you need them [6, 7].
*   While ORMs like **SQLAlchemy** provide robust asynchronous extensions (e.g., `AsyncSession` and `yield_per()` for streaming) [8, 9], you can leverage the underlying driver, such as **`asyncpg`**, to manage cursors directly for maximum performance [7].
*   Postgres requires cursors to be used inside a transaction [10]. Use the cursor's asynchronous generator capabilities (`async for`) to loop through records efficiently, or manually advance the cursor using `.forward()` to skip records instead of using slow SQL `OFFSET` commands [10-12].

---

## Code Examples

Below are best-practice examples demonstrating how to construct robust, memory-safe pagination.

**1. Time-Based Cursor API Response Design**
```json
{
    "next": "https://api.example.com/items?updated_since=2026-03-14T12:00:00Z&size=100",
    "previous": null,
    "result": [
        {
            "id": 105,
            "name": "New Item",
            "timestamp": "2026-03-14T12:00:00Z"
        }
    ]
}
```

2. Asynchronous Database Cursors (asyncpg) To avoid memory exhaustion, this example demonstrates skipping and fetching exact page sizes using a database-level cursor rather than pulling all data into the application.

```python
import asyncpg
import asyncio

async def fetch_paginated_products(offset: int, limit: int):
    # Establish connection (in a real app, acquire this from a connection pool)
    connection = await asyncpg.connect(
        host='127.0.0.1', port=5432, user='postgres', database='products'
    )
    
    # Postgres requires cursors to be created within a transaction block
    async with connection.transaction():
        query = 'SELECT product_id, product_name FROM product ORDER BY created_at ASC'
        
        # 1. Create the cursor pointer
        cursor = await connection.cursor(query)
        
        # 2. Simulate OFFSET: Move the cursor forward efficiently without loading data
        if offset > 0:
            await cursor.forward(offset)
            
        # 3. Simulate LIMIT: Fetch exactly the page size needed
        products_page = await cursor.fetch(limit)
        
        for product in products_page:
            print(f"Retrieved: {product['product_name']}")
            
    await connection.close()

# Example usage: skip the first 500 records and fetch the next 100
# asyncio.run(fetch_paginated_products(offset=500, limit=100))
3. Streaming Large Result Sets with async for If you need to process a massive dataset iteratively without loading it all into memory, use the cursor as an asynchronous generator.
async def stream_all_products(connection: asyncpg.Connection):
    async with connection.transaction():
        query = 'SELECT product_id, product_name FROM product'
        
        # Iterates lazily over the dataset (pre-fetching in chunks automatically)
        async for product in connection.cursor(query):
            # Process one product at a time
            yield product
```