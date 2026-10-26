---
name: turingdb
description: >
  Start, query, write, and manage TuringDB (v3) columnar graph databases using the Python SDK and Cypher.
  TRIGGER when: code imports `turingdb` or `TuringDB`; user mentions TuringDB, turing db, or turing database;
  user asks about TuringDB Cypher queries, graph versioning with changes/commits, vector search in TuringDB,
  GNN training / neighbourhood sampling from TuringDB (DGL, PyTorch, GraphSAGE),
  or the `turingdb` CLI; files contain TuringDB connection strings (localhost:6666) or TuringDB SDK calls
  (`client.query`, `client.new_change`, `client.load_graph`).
  SKIP: generic graph/Neo4j/Cypher questions with no TuringDB mention; general database work unrelated to TuringDB.
---

# TuringDB

TuringDB is a columnar graph database with git-like versioning (changes, commits, time travel). **TuringDB v3** (`turingdb` 3.0.2 on PyPI) runs every query on a new MLIR-based engine and supports most of openCypher:
- **Clauses:** OPTIONAL MATCH, WITH, UNWIND, UNION, MERGE, DETACH DELETE, REMOVE and FOREACH.
- **Patterns:** label expressions (`:A|B`, `:!A`, `:A&B`), and variable-length and quantified path patterns.
- **Expressions:** CASE, comprehensions and `reduce()`.
- **Subqueries:** EXISTS, COUNT, COLLECT and CALL.
- **Query composition:** `WHEN … THEN … ELSE` conditional queries and `NEXT`.
- **Functions:** the common string and math functions.

Aggregation uses implicit grouping, and there are DateTime/Duration types. The Python SDK's `query()` returns pandas DataFrames.

## Setup

**Default: connect to a running TuringDB server over HTTP.** The server listens on `http://localhost:6666` by default. If it isn't running, start one with the `turingdb` CLI (see `startup.md`).

```python
from turingdb import TuringDB          # pip/uv package: turingdb>=3.0.2

client = TuringDB(host="http://localhost:6666")   # HTTP/JSON client (the default backend)
client.create_graph("my_graph")  # create a new graph
# or
client.load_graph("my_graph")    # load an existing on-disk graph by name
client.set_graph("my_graph")     # make it active
```

`TuringDB(...)` has three backends, chosen with `type`:
- `"json"`: HTTP, the default. Pass `host=`.
- `"native"`: binary protocol. Needs a server started with `USE_TURING_PROTO=1`.
- `"embedded"`: in-process, no server. Pass `data_dir=`.

Use the **server** (`json`) by default. If the server was started with `-auth-on`, pass `token=` (or set `TURINGDB_AUTH_TOKEN`). See `startup.md`.

## TuringDB Cypher vs Neo4j Cypher: what to know before writing queries

- **Query parameters only name labels and edge types.** `client.query(q, {"lbl": "Person", "t": "KNOWS"})` fills `MATCH (n:$lbl)-[r:$t]->()` and `CREATE (n:$lbl)`. A `$x` used as a **value** (in WHERE, a property map, SET or RETURN) still fails with "Not implemented: Parameters", so inline literal values into the query text and escape quotes.
- **Bulk data: `LOAD PARQUET` builds graphs, ID-pair `UNWIND` adds edges.**
  - **New graph:** above roughly 5,000 nodes or 1,000 edges, write the data to Parquet and run `LOAD PARQUET`. It builds a new graph in well under a second (see `importing.md`).
  - **Edges between existing nodes:** batch **native node ID pairs** into one `UNWIND`: `UNWIND [[12, 34], …] AS p MATCH (a) WHERE a = p[0] MATCH (b) WHERE b = p[1] CREATE (a)-[:R]->(b)`. That creates 10,000 edges in about 45 ms (see `writing.md`).
  - **Never** match edge endpoints by a key property per row (`{pid: p[0]}`). Each row scans the whole label, so 1,000 pairs take about 7.5 s on 200k nodes.
  - **Embeddings for existing nodes:** use `LOAD EMBEDDING` from a Parquet file, not `SET` loops (see `algorithms.md`).
- **Writes need a change.** Run them as: `client.new_change()` → writes → `CHANGE SUBMIT` → `client.checkout()`. Inside a change, nodes created by an earlier query are invisible to a later **MATCH** until you run `COMMIT`. MERGE does see them. See `writing.md`.
- **Address known nodes by native node ID:** `MATCH (n) WHERE n = 123` (or `id(n) = 123`) is a constant scan of exactly that node.
  - **Batches:** `UNWIND <list of IDs> AS x MATCH (n) WHERE n = x` fetches each node directly, at about 3 µs per node. The list can be a literal, a `WITH` variable, `collect(...)` or `range(...)`, and IDs from the same row (`a = p[0] AND b = p[1]`) work too.
  - **Slow forms:** `WHERE n IN [...]` scans every node. Looking nodes up through a user-level key property per row (`UNWIND keys … {pid: x}`) is about 100× slower than by ID. See `querying.md`.
- **`RETURN n` returns an integer node ID, not a property map.** Use `properties(n)` for the map, `keys(n)` for the property names, or return the properties you need (`n.name, n.age`). Map projection `n{.name}` / `n{.*}` is not supported.
- **Function library gaps.** The common string and math functions exist: `toUpper`, `substring`, `split`, `replace`, `trim`, `left`, `right`, `abs`, `round`, `floor`, `sqrt`, `log` and more, plus `reduce()` and `stDev`/`percentileCont`. Still missing: `=~` regex, `rand()`, `date()`, `timestamp()`, `elementId()`, `round(x, digits)` and `/* */` comments. See `querying.md` for the full list.
- **Reading maps:** `m.key`, `m[k]` with a variable key, and `properties(n).name` all work. A literal key in brackets (`m['key']`) is rejected; write `m.key`. Map values are type-erased: wrap them in `toString()` / `toInteger()` / `toFloat()` before writing them as properties.
- **Labels are fixed at creation.** `SET n:Label` and `REMOVE n:Label` are unsupported. Every node needs at least one label, and every edge exactly one type.
- **Shortest path:** only TuringDB's statement `shortestPath(a, b, weightProp, dist, path)` exists. `shortestPath((a)-[*]-(b))` does not. See `algorithms.md`.
- **Numeric equality.** `Integer = Double` compares exact values (`n.age = 30.0` matches 30). `Double = Double` is still rejected: use `<` / `>` ranges.
- **Function names are case-insensitive** (`AVG` = `avg`), and so are keywords. Labels, edge types and property names are case-sensitive.
- **One statement per request.** `;`-separated scripts are rejected. Comments are `//` only; `--` and `/* */` are not comments.

## Routing

Based on what the user is asking, immediately read the matching file from this same directory using the Read tool. Do not ask the user which file to read — determine it from context.

| Task | File |
|------|------|
| Install, start/stop a server, connect (HTTP, auth, embedded), load/create a graph | `startup.md` |
| Reading data: MATCH, label expressions, OPTIONAL MATCH, WHERE, paths, WITH, NEXT, WHEN, aggregation, UNWIND, maps, subqueries, functions, parameters, LOAD CSV | `querying.md` |
| Writing data: CREATE, MERGE, SET, REMOVE, DELETE, FOREACH, bulk edges by native ID, the change/commit workflow, indexes | `writing.md` |
| Importing external data: CSV, JSONL, GML, Parquet (`LOAD PARQUET` and `turing-parquet`), Neo4j migration | `importing.md` |
| Paths and algorithms: shortest path, variable-length paths, vector/embedding search, writing embeddings back (`LOAD EMBEDDING`), distributed GNN training with `gnn.graphSAGE` sampling | `algorithms.md` |
| Exploring an unfamiliar graph, procedures, versioning/time travel, SDK reference | `introspection.md` |

If the task spans multiple areas (e.g. connect then query), read the files in sequence.
