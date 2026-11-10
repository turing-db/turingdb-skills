---
name: turingdb-introspection
description: Use when exploring an unfamiliar TuringDB graph, listing procedures, inspecting schema, navigating version history, time-travelling to past commits, or looking up the Python SDK API and result types.
---

# TuringDB: Introspection & Versioning

## Exploring an Unfamiliar Graph

Before writing queries against a graph you don't know, use these `CALL` procedures to understand its shape:

```cypher
CALL db.labels()                  // all node labels (yields: id, label)
CALL db.edgeTypes()               // all edge types (yields: id, edgeType)
CALL db.propertyTypes()           // property keys + value types (yields: id, propertyType, valueType)
CALL db.showIndexes()             // property indexes (yields: name, size)
CALL db.history()                 // commit history (yields: commit, nodeCount, edgeCount, partCount)
CALL db.procedures()              // all procedures (yields: name, signature); same as SHOW PROCEDURES
```

Plain Cypher works well for profiling too:

```cypher
MATCH (n) RETURN labels(n) AS labels, count(*) AS n ORDER BY n DESC                 // label combinations
MATCH (a)-[r]->(b) RETURN labels(a) AS src, type(r) AS rel, labels(b) AS dst, count(*) AS n ORDER BY n DESC  // schema triples
MATCH (n:Person) RETURN n.name, n.age LIMIT 5                                       // sample rows
MATCH (n:Person) RETURN count(n.email) AS withEmail, count(*) AS total              // property coverage
```

`RETURN n` gives only an internal ID. To see every property of some nodes, return `n {.*}` or `properties(n)`, which come back as Python dicts:

```cypher
MATCH (n:Person) RETURN id(n), labels(n), n {.*} AS props LIMIT 5
MATCH (n:Person) UNWIND keys(n) AS k RETURN k, count(*) AS n ORDER BY n DESC            // property coverage per key
```

The procedures `db.getNodes` and `db.listNodes` also return properties, as a **JSON string**, plus edge counts. `db.listNodes` adds a case-insensitive substring search:

```cypher
CALL db.getNodes([0, 1, 2]) YIELD id, labels, inEdgeCount, outEdgeCount, properties
CALL db.listNodes(["Person"], {name: "ali"}, 0, 10) YIELD id, labels, properties     // case-insensitive substring search
```

```python
import json
df = client.query("CALL db.getNodes($ids) YIELD id, labels, properties", {"ids": [0, 1, 2]})
df["properties"] = df["properties"].map(json.loads)
```

### All built-in procedures

| Procedure | Yields |
|-----------|--------|
| `db.labels()` | `id, label` |
| `db.edgeTypes()` | `id, edgeType` |
| `db.propertyTypes()` | `id, propertyType, valueType` |
| `db.showIndexes()` | `name, size` |
| `db.history()` | `commit, nodeCount, edgeCount, partCount` |
| `db.describeCommit(commit)` | `nodeCount, edgeCount, partCount` |
| `db.procedures()` | `name, signature` |
| `db.hierarchicalLabelCounts(labels :: LIST)` | `label, nodeCount` |
| `db.listNodes(labels :: LIST, properties :: MAP, skip, limit)` | `id, labels, properties` (JSON string) |
| `db.getNodes(nodeIDs :: LIST)` | `id, labels, inEdgeCount, outEdgeCount, properties` |
| `db.getEdges(edgeIDs :: LIST)` | `id, src, tgt, edgeTypeID, properties` |
| `db.getNodeEdges(nodeIDs, defaultLimit, outLimitTypes, outLimitValues, inLimitTypes, inLimitValues, returnOnlyIDs)` | `id, outgoingEdges, incomingEdges, outEdgeCounts, inEdgeCounts` |
| `gnn.neighbourhoodSample(node, sampleSize, seed = null)` | `src, edge, edgeType, tgt`; samples in-edges (see `algorithms.md`) |
| `gnn.graphSAGE(seeds, fanouts, seed = null)` | `dst_nodes0..2, src_nodes0..2, tgt_nodes0..2`; 3-hop sampler for distributed GNN training (see `algorithms.md`) |

### CALL ... YIELD — chaining procedures into queries

A procedure's output columns can be `YIELD`ed, renamed, filtered and used by later clauses:

```cypher
CALL db.propertyTypes() YIELD *
CALL db.labels() YIELD label AS l WHERE l STARTS WITH 'Gene' RETURN l
CALL db.labels() YIELD label MATCH (n) WHERE label IN labels(n) RETURN label, count(n)
MATCH (n:Station {name: 'Pershore'}) CALL gnn.neighbourhoodSample(n, 5) YIELD src RETURN src.name
```

**Gotchas:**
- **Constant arguments for `db.*` procedures.** An argument can be a literal or a parameter (`db.getNodes($ids)`), but it can't read row values: `db.getNodes([id(n)])` fails with "Argument 'nodeIDs' of 'db.getNodes' must be constant, so it cannot read a row". The `gnn.*` procedures do read their arguments from each row.
- **`valueType` has its own enum type.** `WHERE valueType = 'String'` is rejected ("Operands are not valid or compatible types: 'ValueType' and 'String'"), so filter the DataFrame in pandas instead.
- **`commit` works as a name:** `CALL db.history() YIELD commit` (backticks are still accepted).
- **No `OPTIONAL CALL`** on procedures ("OPTIONAL CALL not supported").

### Extensions

Extensions are native libraries that register extra procedure namespaces:

```cypher
INSTALL greeter                              // load an extension by name (yields extensionName)
SHOW EXTENSIONS                              // list installed extensions (yields name)
SHOW PROCEDURES                              // list all procedures, including extension ones
CALL greeter.hello() YIELD message RETURN message
```

The `greeter` example ships with the pip wheel. Extensions are looked up in `<install>/lib/turingdb/extensions/<name>.so`, then in `<turing-dir>/extensions/<name>.so`.

## Graph and Change Management

```cypher
LIST GRAPH              // graphs currently LOADED in the server (graphName)
LIST AVAILABLE GRAPHS   // graphs on disk, plus in-memory and still-loading ones (graphName, isLoaded, isLoading)
CREATE GRAPH g          // new empty graph
LOAD GRAPH g            // load an on-disk graph
CHANGE LIST             // open changes on the current graph
EXPLAIN <query>         // show the compiled plan without running it; EXPLAIN (db) <query> for the db stage only
```

A query on a graph that isn't loaded fails with `GRAPH_NOT_FOUND`, and the details name the graph.

```python
client.list_available_graphs()  # → list[str]  (any backend)
client.list_loaded_graphs()     # → list[str]
```

There is no `DROP GRAPH` or unload command.

## Versioning / Time Travel

TuringDB stores an immutable commit history. You can query any past state without affecting the current graph.

```python
hist = client.query("CALL db.history()")                    # newest first
head = hist.loc[0, "commit"].removesuffix("(HEAD)")        # HEAD row carries a literal "(HEAD)" suffix

# Query a specific past commit (read-only; writes are refused there)
client.checkout(commit="5007dcb2a08e3011")    # or client.set_commit(...)
df = client.query("MATCH (n) RETURN count(n)")

client.checkout()                 # back to main/HEAD
client.set_change(change_id)      # point at an open change without creating one
client.set_graph("other_graph")   # switch graph context
```

**About `db.history()`:**
- Every `COMMIT` and every submitted change adds an entry.
- `nodeCount` / `edgeCount` count what **that commit's data part** added. They are not graph totals.
- Use hashes exactly as printed. They are not zero-padded.
- `db.describeCommit('<hash>')` returns 0/0/0 for an unknown hash rather than an error.
- An unknown hash passed to `checkout(commit=…)` gives `COMMIT_NOT_FOUND: Commit '<hash>' does not exist`.

A past commit must be loaded into memory before it can be queried. `client.checkout(commit=…)` and the shell's `checkout <commit>` load it for you; `client.set_commit(…)` doesn't, so run this first:
```cypher
LOAD COMMIT 'abc123'
```
A commit that exists but isn't loaded fails with `COMMIT_NOT_LOADED`.

## SDK Client Backends

`TuringDB(...)` is a facade over two interchangeable backends, selected with `type`:

```python
from turingdb import TuringDB

# Default: a running server, over the binary protocol
client = TuringDB(host="localhost", port=6666)                  # type="native"; TuringDB() is the same
client = TuringDB(host="localhost", port=6666, token="s3cret")  # server started with -auth-on

# In-process engine, no daemon/socket — self-contained, no server to manage
client = TuringDB(type="embedded", data_dir="/abs/path/.turing")   # omit data_dir for ~/.turing
```

**Choosing a backend:**
- **`native` (server):** use it by default.
- **`embedded`:** reach for it only when you want a self-contained, in-process engine. It does not support the browser visualizer, concurrent multi-process access, S3 transfers, `host=` or `token=`.
- **Removed in 3.1:** `type="json"`, a URL in `host`, the `HTTPClient` class and the `TURINGDB_TYPE` env var. `token` defaults to `TURINGDB_AUTH_TOKEN`.
- **The query API is identical** on both backends, and results come back as the same DataFrames.

## SDK Method Reference

```python
# Construction: connects at once and raises TuringDBException if no server answers
TuringDB(type="native", host="localhost", port=6666, token=None)   # or type="embedded", data_dir=...

# Query execution; query_parameters fills $name parameters (values, labels, edge types)
client.query(q: str, query_parameters: dict | None = None) -> pandas.DataFrame   # parsed, typed DataFrame
client.query_raw(q: str, query_parameters: dict | None = None) -> dict
#   {"data": {column: values}, "dtypes": {column: type name}, "time": ms}; values are numpy arrays,
#   or lists for strings, nullable columns, lists and maps

# Graph management
client.list_available_graphs() -> list[str]              # any backend
client.list_loaded_graphs() -> list[str]
client.is_graph_loaded() -> bool
client.create_graph(graph_name: str)
client.load_graph(graph_name: str, raise_if_loaded: bool = True)
client.get_graph() -> str
client.set_graph(graph_name: str)

# Change workflow
client.new_change() -> int                               # CHANGE NEW; ALSO switches the client onto it
client.checkout(change: int | "main" = "main", commit: str = "HEAD")   # call after CHANGE SUBMIT
client.set_change(change: int | str)                     # int or decimal string, as CHANGE NEW returns it
client.set_commit(commit: str)
# There is no submit() method: submit via client.query("CHANGE SUBMIT"), then client.checkout()

# Connection / timing
client.reconnect()                                       # new connection; back on the default graph, main, HEAD
client.try_reach(timeout=5)                              # raise if the server doesn't answer within timeout s
client.warmup(timeout=5)
client.get_query_exec_time() -> float | None             # server-side, ms
client.get_total_exec_time() -> float | None             # round-trip, ms

# S3 / data transfer (server backend only)
client.s3_connect(bucket_name, access_key=None, secret_key=None, region=None, use_scratch=True)
client.transfer(src, dst)                                # local / s3:// / turingdb:// paths

# Properties
client.current_graph        # active graph
client.current_commit       # "HEAD" by default
client.current_change       # "main" by default
```

### Result types in the DataFrame

- **Nodes and edges** (`RETURN n`, `RETURN r`) come back as integer IDs (`UInt64`), and as null when an OPTIONAL MATCH didn't match. Use a map projection (`n {.*}`) or `properties(n)` (a dict) for their properties.
- **Scalars:** strings → `string`, Int64 / UInt64 → nullable `Int64` / `UInt64`, Double → `float64`, Bool → `boolean`.
- **Temporal:** DateTime → `datetime64[us, UTC]`, Duration → `timedelta64[us]`. Inside a list or a map they come back as a time-zone-aware `datetime.datetime` or a `datetime.timedelta`.
- **Collections:** lists → Python lists, maps → dicts. Embeddings → numpy float32 arrays.
- **Points:** `turingdb.Point(srid, x, y, z)` named tuples (`z` is None in 2D) with a `crs` property such as `'wgs-84'`.
- **Paths:** a named path `p` and a `shortestPath` `path` both come back as a flat list of IDs, node and edge alternating: `[0, 0, 1]` is node 0, edge 0, node 1.
- **Mixed types:** a column whose rows hold different value types comes back as `object`.
- **Precision:** doubles keep full precision. NaN and ±Infinity come back as float `nan` / `inf`.
- **Integer columns are numpy-backed.** Values pulled out of a DataFrame (`df["id"].iloc[0]`, `itertuples()`) are `np.int64`, which the SDK refuses as a parameter ("A parameter cannot hold a Python int64"). Convert with `.tolist()` or `int()` first.

### Errors

Everything raises `TuringDBException`: query errors, connection errors and protocol errors.
- **Query errors:** `TuringDBException("<CODE>: <details>")`.
  - **Codes:** `PARSE_ERROR`, `ANALYZE_ERROR`, `PLAN_ERROR`, `EXEC_ERROR`, `GRAPH_NOT_FOUND`, `CHANGE_NOT_FOUND`, `COMMIT_NOT_FOUND`, `COMMIT_NOT_LOADED`, ….
  - **Details:** for query errors they include a caret pointing at the offending part of the query. For the `*_NOT_FOUND` codes they name the missing graph, change or commit.
- **Connection errors:**
  - `Failed to connect to localhost:6666: Connection refused` when no server is listening.
  - `Protocol error from server: …` when the server refuses a request: a bad token, or a request over 1 MiB. The server closes the connection, so call `client.reconnect()` and `set_graph()` again.
- **`The query parameters exceed 1048576 bytes`:** the SDK refused the parameters before sending. That client then fails every later parameterized query with `Unknown CellTag.` (and can segfault), even after `reconnect()`. Create a new client, and keep batches smaller.
