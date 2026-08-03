---
name: database-duckdb-spatial-extension

description: Guides the agent through DuckDB's spatial extension for geospatial analysis — converting latitude/longitude to a Point with ST_Point/ST_AsText (and the longitude-first ordering gotcha), finding nearby locations with ST_DWithin, ranking by distance with ST_Distance, the degrees-are-not-a-real-distance-unit caveat on WGS84 data, and briefly handing a result off to a Python mapping/charting library.
---

# DuckDB Spatial Extension for Geospatial Analysis

Beyond reading Excel files (see **database-duckdb-excel-io**), the `spatial`
extension gives DuckDB proper geospatial functions — representing coordinates as
geometry objects and querying them by proximity and distance, directly in SQL.

## 1. Installing and Loading

Same one-time-per-installation pattern as any other extension:

```python
conn.execute("INSTALL spatial")
conn.execute("LOAD spatial")
```

## 2. Converting Coordinates to a Point

`ST_Point(x, y)` builds a geometry from two coordinates; `ST_AsText(...)` renders
it as WKT (well-known text), a human-readable geometry format:

```python
conn.execute("""
    CREATE OR REPLACE TABLE airports_geo AS
    SELECT *, ST_AsText(ST_Point(longitude, latitude)) AS geometry
    FROM airports
""")
```

**Ordering gotcha:** `ST_Point` takes `(x, y)` — that's **longitude, then
latitude** (X is the horizontal/east-west axis) — the opposite of how most people
say a coordinate pair out loud ("latitude, longitude"). Swapping the two silently
produces a valid-looking point in the wrong place rather than an error, since both
are just floats to the function. Double-check this order against the column
names, not against habit.

To parse a WKT string back into a geometry for comparison (e.g. a literal point
you're searching around), use `ST_GeomFromText(...)` — the inverse of
`ST_AsText`.

## 3. Finding Points Within a Distance

`ST_DWithin(geom1, geom2, distance)` tests whether two geometries are within a
given distance of each other:

```python
miami_lng, miami_lat = -80.2706578, 25.7824017

conn.sql(f"""
    SELECT *
    FROM airports_geo
    WHERE ST_DWithin(
        ST_GeomFromText(geometry),
        ST_GeomFromText('POINT ({miami_lng} {miami_lat})'),
        3
    )
""").pl()
```

**Units caveat — this is not real-world distance.** For plain WGS84 coordinates
(`EPSG:4326`, the usual latitude/longitude system), `distance` here is in
*degrees*, not miles or kilometers — and a degree of longitude isn't a fixed
real-world distance, it shrinks the further you get from the equator, while a
degree of latitude is roughly constant. `3` degrees is a rough, direction-biased
approximation good enough for "somewhere in this general area," not for a
distance figure you'd show a user or use for billing/SLA logic. For an accurate
real-world radius search, reproject into a metric coordinate system first, or use
a proper geodesic distance function — don't take a raw-degree `ST_DWithin`
result at face value as "N degrees ≈ N units of real distance."

## 4. Ranking by Distance

`ST_Distance(geom1, geom2)` computes a distance value (same degree-based caveat
as above) you can sort and limit on to find the *closest* matches, rather than
just "within some radius":

```python
conn.sql(f"""
    SELECT *, ST_Distance(
        ST_GeomFromText(geometry),
        ST_GeomFromText('POINT ({miami_lng} {miami_lat})')
    ) AS distance
    FROM airports_geo
    ORDER BY distance
    LIMIT 3
""").pl()
```

This is the same `ORDER BY ... LIMIT N` top-N shape covered in
**database-duckdb-aggregation** section 2 — the same silent-tie caveat applies if
multiple points are equidistant.

## 5. Handing a Result Off to a Mapping/Charting Library

Once a query returns latitude/longitude (or a WKT `geometry` column), plotting it
is no longer a DuckDB concern — convert the result and hand it to whatever
mapping or charting library the task calls for (e.g. `folium` for an interactive
map, `matplotlib`/`seaborn` for a chart):

```python
nearby = conn.sql("SELECT airport, latitude, longitude FROM airports_geo LIMIT 3").pl()

import folium
m = folium.Map(location=[nearby["latitude"][0], nearby["longitude"][0]], zoom_start=6)
for row in nearby.iter_rows(named=True):
    folium.Marker([row["latitude"], row["longitude"]], popup=row["airport"]).add_to(m)
```

Note the one library-specific exception here: Python's GIS stack (`geopandas`,
`shapely`) is built on **pandas**, not Polars — there's no mature Polars-native
equivalent — so a `.to_pandas()` conversion at that specific boundary is
legitimate (see **python-polars-vs-pandas**'s "third-party library that strictly
requires pandas" exception), not a reason to use pandas anywhere else in the
pipeline.

## Related guidance

- **database-duckdb-excel-io** — the `spatial` extension's other use (`st_read` for Excel files), installed the same way.
- **database-duckdb-aggregation** — the `ORDER BY` + `LIMIT` top-N pattern reused in section 4, and its tie caveat.
- **python-polars-vs-pandas** — the general rule the pandas conversion in section 5 is a scoped exception to.
