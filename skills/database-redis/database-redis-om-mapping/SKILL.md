---
name: database-redis-om-mapping

description: Guides the agent on using Redis OM (Python's redis-om + Pydantic, JavaScript's redis-om) to map application objects to Redis JSON/Hash documents declaratively — schema definition with indexed fields, migrations, and expression-based querying — instead of hand-writing HSET/JSON.SET and FT.CREATE/FT.SEARCH calls.
---

# Redis OM (Object Mapping)

You are an expert in Redis OM, the object-mapping framework for Redis. When a user is modeling application entities in Python or JavaScript/TypeScript and wants an object-oriented interface over Redis JSON/Hash storage and RediSearch indexing — instead of hand-writing `HSET`/`JSON.SET` calls and `FT.CREATE`/`FT.SEARCH` strings — guide them to Redis OM and its schema/repository/entity model.

- **Redis OM is a higher-level wrapper over two things this package already covers separately: document storage (`database-redis-json-documents`/Hash via `database-redis-data-modeling-fundamentals`) and RediSearch indexing (`database-redis-search-indexing`).** It doesn't add new server-side capability — it generates the same `JSON.SET`/`HSET` and `FT.CREATE`/`FT.SEARCH` calls for you from a declarative model definition, trading some flexibility for less boilerplate and compile-time-ish structure. Reach for it when an entity's shape is stable enough to declare as a schema; reach for the raw client (and the skills above) when the shape is too dynamic or the query doesn't fit the expression API.
- **Choose JSON-backed models over Hash-backed models the same way you'd choose between the two storage types directly** (see `database-redis-json-documents`): a `JsonModel` supports nested objects/arrays (an `EmbeddedJsonModel` for a sub-object like an address), while a `HashModel` is restricted to flat string/number/binary fields. Model an entity with any nested structure as JSON; a flat entity can use either, but Hash is the lighter-weight choice if there's no nesting.
- **Mark exactly the fields you'll actually query or sort by as indexed** (`Field(index=True)` in Python, the schema's field types in JS) — this is what generates the underlying RediSearch schema. Fields left unindexed are still stored and retrievable by ID, they just can't be used in a `find()`/`search()` filter. Add `full_text_search=True` (Python) or a `text` field type (JS) specifically for free-text fields, the same `TEXT` vs `TAG`/`NUMERIC` distinction covered in `database-redis-search-indexing` — indexing every field as full-text when most are really exact-match/categorical wastes index size and gives worse precision.
- **Indexes must be built explicitly before they can be queried** — Python's `Migrator().run()`, JavaScript's `repository.createIndex()`. Forgetting this step is the most common reason a freshly defined model's `find()`/`search()` returns nothing: the documents exist, but the index that would make them queryable was never created. Run it once at application startup (or as an explicit deploy/migration step), not per-request.
- **A connection must be attached to the model, not passed per-call** — Python does this via each model's `class Meta: database = redisClient`, JavaScript via constructing a `Repository(schema, redisClient)`. This is a different connection-wiring pattern than the raw client libraries in `database-redis-environment-setup`, so don't expect the same connection object to be usable both ways without this extra step.
- **Python's expression-based querying (`Employee.find((Employee.age >= 35) & (Employee.age <= 45))`) compiles down to the same filter/range/full-text query types RediSearch supports** — a numeric range becomes a `NUMERIC` filter, `<<` (containment) against a list field becomes a `TAG` match, `%` becomes full-text search. Understanding the RediSearch layer underneath (`database-redis-search-indexing`) makes it much easier to predict what a given expression will actually do and why a query returns nothing (usually: the field wasn't indexed, or wasn't indexed as the right type).
- **JavaScript/TypeScript support (as of this material) is comparable for JSON modeling and search but has no equivalent for Go** — if a project spans multiple languages and one of them is Go, plan on the raw `go-redis` client plus manual `FT.CREATE`/`FT.SEARCH` (see `database-redis-search-indexing`) for that service, not Redis OM; don't assume feature parity across every language this package covers.
- **An entity gets an auto-generated unique ID on save** (a ULID in Python's `redis-om`) — this becomes part of the key Redis OM stores the document under. Capture and store this ID (`new_employee.pk` in Python, `entity[EntityId]` in JS) if the caller needs to look the entity up again later — it's not derived from any field you supplied.

## Code Examples

**Python (redis-om + Pydantic)**

```python
from typing import List
from redis_om import EmbeddedJsonModel, Field, JsonModel, get_redis_connection, Migrator

redis_conn = get_redis_connection(host="127.0.0.1", port=6379, decode_responses=True)

class Office(EmbeddedJsonModel):
    city: str = Field(index=True)
    country: str = Field(index=True, default="Remote")
    class Meta:
        database = redis_conn

class Employee(JsonModel):
    firstname: str = Field(index=True)
    lastname: str = Field(index=True)
    age: int = Field(index=True)
    office: Office
    roles: List[str] = Field(index=True)
    fun_fact: str = Field(index=True, full_text_search=True)
    class Meta:
        database = redis_conn

Migrator().run()  # builds the RediSearch index — required before find()/search() work

async def create_employee() -> Employee:
    employee = Employee(
        firstname="Luigi", lastname="Fugaro", age=44,
        office=Office(city="Roma", country="Italy"),
        roles=["Solution Architect"], fun_fact="No matter what, we are all in sales!",
    )
    employee.save()
    return employee  # employee.pk is the generated ID

def find_by_age_range(min_age: int, max_age: int) -> list[Employee]:
    return Employee.find((Employee.age >= min_age) & (Employee.age <= max_age)).sort_by("age").all()

def find_by_role_and_city(role: str, city: str) -> list[Employee]:
    return Employee.find((Employee.roles << role) & (Employee.office.city == city)).all()
```

**JavaScript/TypeScript (redis-om)**

```typescript
import { createClient } from "redis";
import { Schema, Repository, EntityId } from "redis-om";

const redisClient = createClient({ socket: { host: "127.0.0.1", port: 6379 } });
await redisClient.connect();

const employeeSchema = new Schema(
  "Employee",
  {
    firstName: { type: "string" },
    lastName: { type: "string" },
    age: { type: "number" },
    roles: { type: "string[]" },
    office: { type: "text" },
    funFact: { type: "text" },
  },
  { dataStructure: "JSON" },
);

export const employeeRepository = new Repository(employeeSchema, redisClient);
await employeeRepository.createIndex(); // required before search() works

async function createEmployee() {
  const saved = await employeeRepository.save({
    firstName: "Luigi",
    lastName: "Fugaro",
    age: 45,
    roles: ["Solution Architect"],
    office: "Roma",
    funFact: "I still play PAC-MAN.",
  });
  return saved[EntityId]; // the generated entity ID
}

async function findAllEmployees() {
  return employeeRepository.search().return.all();
}
```
