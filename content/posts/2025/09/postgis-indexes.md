---

title: "Speeding Up Geospatial Queries with PostGIS GiST Indexes"
date: 2025-09-12T07:00:00-07:00
draft: false
categories: \[PostGIS, Database, Geospatial]
--------------------------------------------

When working with geospatial data, performance can get slow as your dataset grows. PostGIS provides a tool to make these queries fast: the **GiST index**.

### Example Problem

Suppose you have **weather stations** (points) and **city boundaries** (polygons). You want to find which stations are inside a city:

```sql
SELECT s.id
FROM stations s
JOIN cities c ON ST_Within(s.location, c.boundary)
WHERE c.id = 42;
```

Without an index, Postgres must check every row, which is slow for millions of stations.

#### Visual Representation of the Tables

**stations table:**

```
 id | name        | location
----+-------------+-----------------------------
  1 | Station A   | POINT(30.5 50.2)
  2 | Station B   | POINT(30.7 50.4)
  3 | Station C   | POINT(31.0 50.6)
```

**cities table:**

```
 id | name      | boundary
----+-----------+-----------------------------------------------
 42 | City X    | POLYGON((30.0 50.0, 31.5 50.0, 31.5 51.0, 30.0 51.0, 30.0 50.0))
 43 | City Y    | POLYGON((32.0 49.5, 33.0 49.5, 33.0 50.5, 32.0 50.5, 32.0 49.5))
```

The query checks which points (stations) fall inside the polygon of City X.

### Using GiST Indexes

A GiST index stores bounding boxes for geometries. Postgres first checks which bounding boxes overlap (fast), then runs the exact `ST_Within` check on a smaller set.

```sql
CREATE INDEX idx_cities_boundary ON cities USING GIST (boundary);
CREATE INDEX idx_stations_location ON stations USING GIST (location);
```

### Measuring Performance

```sql
EXPLAIN ANALYZE
SELECT s.id
FROM stations s
JOIN cities c ON ST_Within(s.location, c.boundary)
WHERE c.id = 42;
```

Before adding an index, using `EXPLAIN ANALYZE` might produce output like this:

```
Seq Scan on stations s  (cost=0.00..13456.00 rows=1000 width=16) (actual time=0.015..3450.320 rows=1024 loops=1)
  Join Filter: ST_Within(s.location, c.boundary)
Planning Time: 0.125 ms
Execution Time: 3450.400 ms
```

After adding a GiST index, the same query might produce output like:

```
Bitmap Heap Scan on stations s  (cost=4.25..45.75 rows=1024 width=16) (actual time=0.023..15.450 rows=1024 loops=1)
  Recheck Cond: (s.location && c.boundary)
  ->  Bitmap Index Scan on idx_stations_location  (cost=0.00..4.20 rows=1024 width=0) (actual time=0.018..0.018 rows=1024 loops=1)
Planning Time: 0.210 ms
Execution Time: 15.500 ms
```

You can see the dramatic reduction in execution time from **\~3.4 seconds** to **\~15 milliseconds**.

### Maintenance

GiST indexes require maintenance to stay efficient:

* **VACUUM ANALYZE:** Reclaims space from deleted or updated rows and updates planner statistics.

  ```sql
  VACUUM ANALYZE stations;
  VACUUM ANALYZE cities;
  ```
* **REINDEX:** Over time, especially with lots of updates, GiST indexes can become bloated. Rebuilding them keeps performance optimal.

  ```sql
  REINDEX INDEX idx_stations_location;
  REINDEX INDEX idx_cities_boundary;
  ```
* **Maintenance Best Practices:** Run `VACUUM ANALYZE` (weekly) and `REINDEX` (quarterly) via scripts or as part of your deployment or migration process. This ensures indexes remain healthy and query performance stays consistent over time.

### Inspecting Indexes

After creating GiST indexes, the table data itself doesn’t change, but you can inspect which indexes exist using Postgres tools:

Example:

```
>> \d stations
Indexes:
    "stations_pkey" PRIMARY KEY, btree (id)
    "idx_stations_location" gist (location)
```

Or query the catalog:

```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'stations';
```

This confirms your GiST indexes are in place.

### Benefits

* **Faster queries:** Milliseconds instead of seconds.
* **Lower DB load:** Can handle more users.

GiST indexes with `ST_Within` are a simple, effective way to scale geospatial queries in Postgres, and regular maintenance keeps performance reliable.
