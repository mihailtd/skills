---
name: database-redis-geospatial

description: Guides the agent on Redis's native geospatial commands (GEOADD/GEODIST/GEOSEARCH) for proximity and radius queries, RediSearch's GEO field type for indexed geo search combined with other filters, and the GEOSHAPE field type for point/polygon area queries (CONTAINS/WITHIN) via Well-Known Text.
---

# Redis Geospatial Indexing

You are an expert in Redis's geospatial data type. When a user needs to store locations and query by proximity or distance ("nearest X," "within N km of here"), guide them to the native `GEO*` commands, and help them choose between that and RediSearch's `GEO` field type based on whether the query needs to combine location with other filters.

- **A geospatial index is a Sorted Set under the hood**, with each member's location encoded into its score via Geohash — this is why geospatial queries get the same low-complexity range-query performance as any other Sorted Set operation, without needing a separate spatial index structure.
- **Add locations with `GEOADD`**, giving longitude before latitude (the reverse of how coordinates are usually spoken/written — a frequent source of bugs):

  ```
  GEOADD Italy 14.166667 42.349998 Chieti
  GEOADD Italy 11.330556 43.318611 Siena
  ```

- **Get the distance between two already-indexed members with `GEODIST`**, specifying the unit explicitly (`m`, `km`, `mi`, `ft`):

  ```
  GEODIST Italy Chieti Siena km
  ```

- **Find everything within a radius of an arbitrary point with `GEOSEARCH ... FROMLONLAT ... BYRADIUS`**, sorted by distance and with distances returned in the same call — no separate pass to compute and sort distances client-side:

  ```
  GEOSEARCH Italy FROMLONLAT 13 43 BYRADIUS 200 km ASC WITHDIST
  ```

  `GEOSEARCH` also supports `BYBOX` (rectangular area) and searching from an existing member instead of raw coordinates (`FROMMEMBER`) — pick the search shape that matches the actual requirement (a radius for "nearby," a box for a map viewport) rather than always defaulting to radius.
- **Use plain `GEO*` commands when location is the only or primary filter.** Use RediSearch's `GEO` field type instead (see `database-redis-search-indexing`) when a query needs to combine location with other attributes in the same call — e.g. "novels within 100km of this point, published after 2010, in stock" — because RediSearch can intersect a geo filter with `TEXT`/`TAG`/`NUMERIC` filters in one indexed query, where the plain `GEO*` commands only ever filter on location:

  ```
  FT.CREATE docs_idx ON HASH PREFIX 1 doc: SCHEMA Name AS name TEXT Location AS location GEO
  FT.SEARCH docs_idx "Wuthering @location:[13.50337 43.5942 100 km]"
  ```

  Don't default to hand-combining a `GEOSEARCH` result with a separate filter step in application code once RediSearch is available — that reintroduces the same "multiple round trips, filter client-side" cost this package generally steers away from (see `database-redis-manual-secondary-indexing`).

## Shapes and Polygons (GEOSHAPE)

- **The plain `GEO` field type (and `GEO*` commands above) only ever model points** — a location is a single lon/lat pair, and every query is "how far is this point from that point" or "what points fall in this radius/box." When the requirement is genuinely about **areas** — does this delivery zone contain that address, does this parcel boundary overlap that one — reach for the `GEOSHAPE` field type instead, which models points *and* polygons in the same index.
- **`GEOSHAPE` requires choosing a coordinate system explicitly at index-creation time**, and the choice isn't just cosmetic — it determines what the numbers in stored geometries mean:
  - **`SPHERICAL`** — real-world longitude/latitude coordinates, for actual geographic areas (delivery zones, service boundaries, districts).
  - **`FLAT`** — plain X/Y coordinates on a Cartesian plane, for non-geographic spatial data (floor plans, game maps, any 2D layout that isn't tied to the Earth's surface).

  ```
  FT.CREATE polygon_idx PREFIX 1 shape: SCHEMA g GEOSHAPE FLAT t TEXT
  ```

- **Geometries are stored as Well-Known Text (WKT)** — a point is `POINT (x y)`, a polygon is `POLYGON((x1 y1, x2 y2, ..., x1 y1))`. **A polygon's ring must be closed**: the first and last coordinate pairs must match, or the geometry is malformed.

  ```
  HSET shape:1 t "this is a triangle" g "POLYGON((2 2, 6 8, 10 2, 2 2))"
  HSET shape:point:1 g 'POINT(3 3)'
  ```

- **Query area relationships with `CONTAINS`/`WITHIN`, passed as a query parameter, not inline in the query string** — geometry WKT strings are long and easy to mis-escape inline, so `PARAMS` is the practical (and idiomatic) way to supply them:

  ```
  FT.SEARCH polygon_idx "@g:[CONTAINS $poly]" PARAMS 2 poly 'POLYGON((6 6, 6 7, 7 7, 7 6, 6 6))' DIALECT 3
  FT.SEARCH polygon_idx "@g:[WITHIN $poly]" PARAMS 2 poly 'POLYGON((13 13, 13 15, 15 15, 15 13, 13 13))' DIALECT 3
  ```

  `CONTAINS` returns indexed geometries that *contain* the query geometry (e.g. "which of my stored zones contains this delivery address point"); `WITHIN` returns indexed geometries that *fall inside* the query geometry (e.g. "which of my stored points fall inside this search area"). These are inverses of each other — picking the wrong one silently returns an empty or wrong result set rather than erroring, so double check which direction the containment actually needs to go before running the query.
- **`GEOSHAPE` queries require `DIALECT 3`** — see `database-redis-search-query-syntax` for why dialect matters and how to set it, either per-query (as shown above) or as the server default via `FT.CONFIG SET DEFAULT_DIALECT 3`.
- This is a comparatively new, still-evolving capability (introduced in Redis Stack 7.2) — confirm which relation operators (`CONTAINS`/`WITHIN`, and any added since) are supported in the Redis Stack version actually deployed before assuming the full set is available.

## Code Examples

```python
from redis import asyncio as aioredis

client = aioredis.from_url("redis://localhost")

async def add_location(index_key: str, name: str, longitude: float, latitude: float) -> None:
    """Longitude first, then latitude — the opposite of how coordinates are usually spoken."""
    await client.geoadd(index_key, (longitude, latitude, name))

async def distance_km(index_key: str, place_a: str, place_b: str) -> float:
    return await client.geodist(index_key, place_a, place_b, unit="km")

async def nearby(index_key: str, longitude: float, latitude: float, radius_km: float) -> list[tuple[str, float]]:
    """Sorted by distance, with distances included — no client-side sort needed."""
    results = await client.geosearch(
        index_key, longitude=longitude, latitude=latitude,
        radius=radius_km, unit="km", sort="ASC", withdist=True,
    )
    return results

async def zones_containing_point(index_name: str, point_wkt: str) -> list[str]:
    """GEOSHAPE: which stored polygons (e.g. delivery zones) contain this point?
    Raw command used here since redis-py's high-level Query API support for
    GEOSHAPE PARAMS varies by version — confirm current client support."""
    result = await client.execute_command(
        "FT.SEARCH", index_name, "@g:[CONTAINS $poly]",
        "PARAMS", 2, "poly", point_wkt, "DIALECT", 3,
    )
    return result
```
