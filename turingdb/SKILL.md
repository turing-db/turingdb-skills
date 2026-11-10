---
name: turingdb
description: >
  Start, query, write, and manage TuringDB (v3.1) columnar graph databases using the Python SDK, the
  JavaScript/TypeScript SDK and Cypher.
  TRIGGER when: code imports `turingdb` or `TuringDB` (Python) or `TuringClient` from the `turingdb` npm package;
  user mentions TuringDB, turing db, or turing database;
  user asks about TuringDB Cypher queries, query parameters, graph versioning with changes/commits, vector search
  or point/distance queries in TuringDB,
  GNN training / neighbourhood sampling from TuringDB (DGL, PyTorch, GraphSAGE),
  or the `turingdb` CLI; files contain TuringDB connection settings (localhost:6666) or TuringDB SDK calls
  (`client.query`, `client.new_change`, `client.load_graph`, `client.queryData`).
  SKIP: generic graph/Neo4j/Cypher questions with no TuringDB mention; general database work unrelated to TuringDB.
---

# TuringDB

TuringDB is a columnar graph database with git-like versioning (changes, commits, time travel). **TuringDB 3.1** (`turingdb` 3.1 on PyPI and npm) runs every query on an MLIR-based engine and supports most of openCypher:
- **Clauses:** OPTIONAL MATCH, WITH, UNWIND, UNION, MERGE, DETACH DELETE, REMOVE, FOREACH, and `SET n = map` / `SET n += map`.
- **Patterns:** label expressions (`:A|B`, `:!A`, `:A&B`), and variable-length and quantified path patterns.
- **Expressions:** CASE, comprehensions, `reduce()`, `allReduce()`, map projection (`n {.name, .age}`) and map access (`m['key']`, `m.a.b`).
- **Subqueries:** EXISTS, COUNT, COLLECT and CALL, each holding any query (UNION, NEXT, WHEN).
- **Query composition:** `WHEN … THEN … ELSE` conditional queries and `NEXT`.
- **Parameters:** `$name` for values, labels and edge types.
- **Types:** DateTime, Duration, Embedding and Point (cartesian or WGS-84, with haversine distances).

Aggregation uses implicit grouping. The Python SDK's `query()` returns pandas DataFrames.

## Setup

**Default: connect to a running TuringDB server.** Clients speak TuringDB's binary protocol over HTTP/1.1, to `localhost:6666` by default. If no server is running, start one with the `turingdb` CLI (see `startup.md`).

```python
from turingdb import TuringDB          # pip/uv package: turingdb>=3.1

client = TuringDB(host="localhost", port=6666)   # or TuringDB(); host is a name or IP, never a URL
client.create_graph("my_graph")  # create a new graph
# or
client.load_graph("my_graph")    # load an existing on-disk graph by name
client.set_graph("my_graph")     # make it active
```

`TuringDB(...)` has two backends, chosen with `type`:
- `"native"`: a running server, over the binary protocol. The default. Pass `host=` and `port=`.
- `"embedded"`: in-process, no server. Pass `data_dir=`.

Use the **server** by default. If it was started with `-auth-on`, pass `token=` (or set `TURINGDB_AUTH_TOKEN`). The JSON/HTTP API of 3.0 is gone: `curl` can't query the server, and `type="json"` no longer exists. For JavaScript or TypeScript, use the `turingdb` npm package (`TuringClient`). See `startup.md`.

## TuringDB Cypher vs Neo4j Cypher: what to know before writing queries

- **Pass values as parameters.** `client.query("MATCH (n:Person) WHERE n.age >= $min RETURN n.name", {"min": 30})`. A `$name` stands for a value anywhere an expression goes, and also for a label or an edge type (`MATCH (n:$lbl)`). Never format values into the query text.
  - **Not yet:** `SKIP $n` / `LIMIT $n` ("Unsupported expression in SKIP/LIMIT: PARAMETER") and the vector of a `VECTOR SEARCH`. Write those as literals.
  - **Size:** parameters count toward the 1 MiB request limit, which is about 38,000 pairs of 7-digit IDs. Batch 10,000–20,000 rows per query. **After the SDK refuses oversized parameters, that client fails every later parameterized query** (`Unknown CellTag.`, and it can segfault). `reconnect()` doesn't fix it, so create a new `TuringDB` client.
- **Parameter lists: two planner traps.** Both look natural and both scan:
  - `MATCH (n) WHERE n IN $ids` scans every node (87 ms for 1,000 IDs on 200k nodes). Use `UNWIND $ids AS x MATCH (n) WHERE n = x` (about 3 ms). A literal `IN [1, 2, 3]` is a direct fetch.
  - `UNWIND $keys AS x MATCH (n:P {pid: x})` (or `$rows` with `r.pid`) scans the label for every row, index or not: 1.7–2.7 s for 1,000 keys on 200k nodes. Map keys to native IDs first, or fetch the nodes with `MATCH (n:P) WHERE n.pid IN $keys` (one scan, 79 ms). `MERGE` by key, LOAD CSV joins and two-endpoint key matches are hash-joined and fast.
- **Bulk data: `LOAD PARQUET` builds graphs, `UNWIND $pairs` adds edges.**
  - **New graph:** above roughly 5,000 nodes or 1,000 edges, write the data to Parquet and run `LOAD PARQUET`. It builds 200k nodes and 400k edges in about 0.3 s (see `importing.md`).
  - **Edges between existing nodes:** batch **native node ID pairs** as one parameter: `UNWIND $pairs AS p MATCH (a) WHERE a = p[0] MATCH (b) WHERE b = p[1] CREATE (a)-[:R]->(b)`. That creates 20,000 edges in about 10 ms (see `writing.md`).
  - **New nodes:** `UNWIND $rows AS row CREATE (:City {name: row.name, pop: row.pop})` writes 20,000 nodes in about 60 ms. Values read from maps and lists are stored as they are, with no `toString()` / `toInteger()` needed.
  - **Embeddings for existing nodes:** use `LOAD EMBEDDING` from a Parquet file, not `SET` loops (see `algorithms.md`).
- **Writes need a change.** Run them as: `client.new_change()` → writes → `CHANGE SUBMIT` → `client.checkout()`. Inside a change, nodes created by an earlier query are invisible to a later **MATCH** until you run `COMMIT`. MERGE does see them. See `writing.md`.
- **Address known nodes by native node ID:** `MATCH (n) WHERE n = 123` (or `id(n) = 123`) is a constant scan of exactly that node. `UNWIND $ids AS x MATCH (n) WHERE n = x` fetches each node directly. Write `n = x`, not `id(n) = x`: against a parameter, the `id(n)` form scans every node (653 ms for 1,000 IDs, against 0.2 ms). Check plans with `EXPLAIN (db)`: `const_scan_nodes` / `fetch_nodes` / `hash_join` are good, while `scan_nodes…` followed by `unwind` or a filter is a scan. See `querying.md`.
- **`RETURN n` returns an integer node ID, not a property map.** Use a map projection (`n {.name, .age}`, `n {.*}`), `properties(n)`, or return the properties you need (`n.name, n.age`). A path comes back as a flat list of IDs, node and edge alternating.
- **Function library gaps.** The common string, math, list and temporal functions exist, as do `elementId`, `isEmpty`, `coll.indexOf`, `point.distance` and `point.withinBBox`. Still missing: `=~` regex, `rand()`, `date()`, `timestamp()`, `atan2()`, `isNaN()` and `round(x, digits)`. `exists(n.prop)` is refused: write `n.prop IS NOT NULL`. See `querying.md` for the full list.
- **Labels are fixed at creation.** `SET n:Label` and `REMOVE n:Label` are unsupported. Every node needs at least one label, and every edge exactly one type.
- **Property types are global.** Each property name has one value type across the graph, and Integer properties don't widen to Double.
- **Shortest path:** only TuringDB's statement `shortestPath(a, b, weightProp, dist, path)` exists. `shortestPath((a)-[*]-(b))` does not. See `algorithms.md`.
- **Numeric equality.** `Integer = Double` compares exact values (`n.age = 30.0` matches 30). `Double = Double` is still rejected: use `<` / `>` ranges.
- **Function names are case-insensitive** (`AVG` = `avg`), and so are keywords. Labels, edge types and property names are case-sensitive. Keywords can be names (`RETURN n.name AS count`).
- **One statement per request.** `;`-separated scripts are rejected; chain steps with `NEXT`. Comments are `//` and `/* */`; `--` is not a comment.

## Routing

Based on what the user is asking, immediately read the matching file from this same directory using the Read tool. Do not ask the user which file to read — determine it from context.

| Task | File |
|------|------|
| Install, start/stop a server, connect (Python native or embedded, auth), the JavaScript/TypeScript client, load/create a graph | `startup.md` |
| Reading data: MATCH, label expressions, OPTIONAL MATCH, WHERE, paths, WITH, NEXT, WHEN, aggregation, UNWIND, maps, subqueries, functions, points, parameters, LOAD CSV | `querying.md` |
| Writing data: CREATE, MERGE, SET (incl. from a map), REMOVE, DELETE, FOREACH, bulk writes with parameters, bulk edges by native ID, the change/commit workflow, indexes | `writing.md` |
| Importing external data: CSV, JSONL, GML, Parquet (`LOAD PARQUET` and `turing-parquet`), Neo4j migration | `importing.md` |
| Paths and algorithms: shortest path, variable-length paths, vector/embedding search, writing embeddings back (`LOAD EMBEDDING`), geographic distances, distributed GNN training with `gnn.graphSAGE` sampling | `algorithms.md` |
| Exploring an unfamiliar graph, procedures, versioning/time travel, SDK reference, result types, errors | `introspection.md` |

If the task spans multiple areas (e.g. connect then query), read the files in sequence.
