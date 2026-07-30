---
name: python-beanie-documents
description: Instructs the agent on modeling MongoDB collections as Beanie Documents (async ODM built on Pydantic and PyMongo's async driver), initialization with init_beanie, indexes, CRUD operations, querying, and Link/BackLink relations between documents.
---

# Python Beanie Document Modeling & CRUD Guidelines

You are an expert Python developer specializing in Beanie, the asynchronous MongoDB ODM built on Pydantic and PyMongo's async driver. When asked to define MongoDB document models, write queries, or wire up relations between collections, adhere to the following rules.

## 1. Document Definition

Subclass `Document` (not Pydantic's `BaseModel` directly) for anything that maps to a MongoDB collection. Every `Document` automatically gets an `id` field backed by MongoDB's `_id` (defaults to `PydanticObjectId`; override with a `Field` annotation if you need a different id type).

- Declare fields exactly like a Pydantic model — types, defaults, validators all work as usual.
- Use the `Settings` inner class to configure the collection name and indexes; without it, Beanie derives the collection name from the class name.

```python
from beanie import Document, Indexed
import pymongo

class Product(Document):
    name: Indexed(str, unique=True)
    description: Indexed(str, index_type=pymongo.TEXT)
    price: float
    category: str

    class Settings:
        name = "products"
        indexes = [
            "category",
            [("price", pymongo.DESCENDING)],
        ]
```

## 2. Initialization

Call `init_beanie()` exactly once at startup, before any `Document` is used. Pass the async database handle and every `Document` subclass. In FastAPI, do this in the lifespan handler, not at import time.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from pymongo import AsyncMongoClient
from beanie import init_beanie

@asynccontextmanager
async def lifespan(app: FastAPI):
    client = AsyncMongoClient("mongodb://user:pass@host:27017")
    await init_beanie(database=client.app_db, document_models=[Product])
    yield

app = FastAPI(lifespan=lifespan)
```

`init_beanie` creates collections if they don't exist and syncs any indexes declared in `Settings`. Never call it more than once per process, and never issue a query before it has completed.

## 3. Inserting Documents

- Instantiate the `Document` subclass like a normal Pydantic model, then call `.insert()` (or its alias `.create()`).
- For multiple documents of the same type, always prefer `Model.insert_many([...])` over a loop of individual `.insert()` calls — it batches the write into a single round-trip.

```python
bar = Product(name="Tony's", price=5.95, category="chocolate")
await bar.insert()

await Product.insert_many([bar1, bar2, bar3])
```

## 4. Querying

- Use `Model.get(id)` to fetch by primary key, `Model.find_one(...)` for a single match, and `Model.find(...)` for multiple matches.
- Filter with plain Python comparison operators on class fields (`==`, `<`, `>=`, etc.), or with `beanie.operators` (e.g. `In`, `And`, `Or`) for MongoDB-specific operators. Raw PyMongo dict syntax also works when nothing else fits.
- `find()` returns a lazy `FindMany` query object — chain `.sort()`, `.skip()`, `.limit()`, `.project()` before awaiting `.to_list()`, iterating with `async for`, or calling `.first_or_none()`.

```python
from beanie.operators import In

product = await Product.get("608da169eb9e17281f0ab2ff")
cheap = await Product.find(Product.price < 10).to_list()
some = await Product.find(In(Product.category, ["chocolate", "fruits"])).to_list()

chocolates = (
    await Product.find(Product.category == "chocolate")
    .sort(-Product.price, +Product.name)
    .skip(0)
    .limit(20)
    .to_list()
)
```

Use `.project(SomeSmallerModel)` to fetch only the fields a caller needs instead of the whole document — cheaper over the wire and avoids leaking internal fields.

## 5. Updating and Deleting

- Prefer `.save()` on an already-fetched instance for simple read-modify-write flows — it inserts if the document doesn't exist yet, updates if it does.
- Prefer targeted update operators (`.set()`, `.inc()`, `.update(Set({...}))`) over fetch-then-save when you don't need the document in memory — it's a single atomic operation instead of a read + write.
- `.upsert()` combines a filtered find with an update-or-insert in one call.

```python
bar = await Product.find_one(Product.name == "Mars")
bar.price = 10
await bar.save()

await Product.find(Product.price > 0.5).inc({Product.price: 1})

await Product.find_one(Product.name == "Tony's").upsert(
    Set({Product.price: 3.33}),
    on_insert=Product(name="Tony's", price=3.33, category="chocolate"),
)

await bar.delete()
await Product.find(Product.category == "chocolate").delete()
```

## 6. Relations — `Link` and `BackLink`

MongoDB has no foreign keys or joins. Beanie's `Link[Model]` stores a DBRef-style reference and fetches it on demand — use it only where a true cross-document reference is needed; prefer embedding for data that's always read together (see the embedding-vs-linking guidance in the **database-mongodb** package for the underlying data-modeling tradeoff).

- Declare a reference field as `Link[Model]`, `Optional[Link[Model]]`, or `List[Link[Model]]`.
- Links are **not** fetched by default — pass `fetch_links=True` to `find()`/`find_one()`/`get()` to resolve them, or call `.fetch_link(...)` / `.fetch_all_links()` on an already-loaded instance.
- Control cascading writes/deletes with `WriteRules`/`DeleteRules` (`WriteRules.WRITE` to insert/replace linked documents together, `DeleteRules.DELETE_LINKS` to cascade a delete).
- For the reverse direction, declare a `BackLink[Model]` on the referenced side (requires `fetch_links=True` and an `original_field` pointing back at the owning field).

```python
from typing import List, Optional
from beanie import Document, Link, BackLink, WriteRules
from pydantic import Field

class Door(Document):
    height: int

class House(Document):
    name: str
    door: Link[Door]
    windows: List[Link["Window"]]

class DoorWithHouse(Document):
    height: int
    house: BackLink[House] = Field(json_schema_extra={"original_field": "door"})

house = House(name="Beach House", door=Door(height=2), windows=[])
await house.save(link_rule=WriteRules.WRITE)  # inserts the linked Door too

houses = await House.find(House.name == "Beach House", fetch_links=True).to_list()
```

## 7. When to reach for raw PyMongo instead

Beanie's abstraction is worth leaving for bulk/aggregation-heavy work — complex `$lookup`/`$facet` aggregation pipelines, large bulk writes, or anything where the ODM layer adds overhead without ergonomic benefit. Drop to the underlying `AsyncMongoClient` (`Model.get_motor_collection()` or your own client handle) for those cases rather than fighting the ODM.
