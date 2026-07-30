---
name: database-mongodb-zone-sharding
description: Guides the agent on MongoDB zone sharding, tag ranges, geographic placement, and hot/cold data segregation.
---

# MongoDB Zone Sharding

You are an expert on MongoDB zone sharding. When asked about zones, tag ranges, or data locality, explain how to use zones to pin ranges of shard key values to specific shards and why this is useful.

## 1. What zone sharding is

- Zone sharding lets you assign ranges of shard key values to named zones.
- Each zone maps to one or more shards.
- The balancer uses tag ranges to ensure documents in a zone reside on the correct shards.
- Zones are useful for geographic locality, hardware tiering, and regulatory isolation.

## 2. Common zone sharding use cases

- Serve US traffic from US-based shards and EU traffic from EU-based shards.
- Keep hot data on fast hardware and cold data on slower, cheaper hardware.
- Satisfy data residency or compliance requirements.
- Provide application-aware query routing when queries are mostly local.

## 3. Basic zone sharding commands

### Add shards to zones

```js
sh.addShardToZone('shardRS2', 'US');
sh.addShardToZone('shardRS', 'TheWorld');
```

### Tag a shard key range

```js
sh.addTagRange(
  'MongoDBTuningBook.customers',
  { Country: 'Afghanistan', City: MinKey },
  { Country: 'United Kingdom', City: MaxKey },
  'TheWorld'
);
sh.addTagRange(
  'MongoDBTuningBook.customers',
  { Country: 'United States', City: MinKey },
  { Country: 'United States', City: MaxKey },
  'US'
);
```

## 4. Zone sharding design tips

- Choose shard key fields that support the zone range semantics, such as `{ country: 1, city: 1 }`.
- Use `MinKey` and `MaxKey` to cover open-ended ranges.
- Avoid overly broad zones that force too much data onto a single shard.
- Keep zone ranges precise enough to express locality or tiering goals without fragmenting the chunk map unnecessarily.

## 5. Balancer and query routing behavior

- Queries that include the zone-aware shard key range can still be routed efficiently.
- Queries for data outside a zone can be routed to the appropriate shards by `mongos`.
- If an application in one zone queries another zone’s data, latency may increase.
- Deploy `mongos` routers close to the application region for the best performance.

## 6. Hot/cold and hardware-tiering strategies

- Place recent, frequently-updated data in a high-performance zone.
- Place archive or cold data in a zone backed by slower disks or smaller instances.
- Use zones to align expensive hardware with hot time windows or tenant segments.

## 7. Managing zones

### Remove a tag range

```js
sh.removeTagRange(
  'MongoDBTuningBook.customers',
  { Country: 'United States', City: MinKey },
  { Country: 'United States', City: MaxKey }
);
```

### Remove a shard from a zone

```js
sh.removeShardFromZone('shardRS2', 'US');
```

### Verify zone assignments

```js
sh.status();
```

and inspect the output for `zone` or `tagRanges` sections.

## 8. Cautions and performance tips

- Zone sharding increases planning complexity and must be justified by locality, cost, or compliance needs.
- Avoid using zone sharding as a band-aid for a bad shard key.
- If tag ranges are too narrow, the balancer may struggle to keep chunks in place.
- Monitor chunk movement and zone alignment after adding or changing tag ranges.

## 9. Answering style

- Describe zone sharding as a deliberate placement strategy, not an automatic balancing feature.
- Use geographic and hot/cold examples to illustrate the benefit.
- Mention that zones are implemented with `sh.addShardToZone()` and `sh.addTagRange()`.
- Warn that cross-zone queries may incur higher latency.
- Explain that zone sharding is most useful when data locality matters.
