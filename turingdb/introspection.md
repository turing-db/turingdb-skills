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

`RETURN n` gives only an internal ID, and there is no `properties(n)`. To see every property of a few nodes, use `db.getNodes`, which returns `properties` as a JSON string:

```cypher
CALL db.getNodes([0, 1, 2]) YIELD id, labels, properties RETURN id, labels, properties
CALL db.listNodes(["Person"], {name: "ali"}, 0, 10) YIELD id, labels, properties     // case-insensitive substring search
```

```python
import json
df = client.query("CALL db.getNodes([0, 1, 2]) YIELD id, labels, properties")
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
- **Constant arguments only.** A procedure argument can't read row values (`db.getNodes([id(n)])` fails with "must be constant").
- **`valueType` has its own enum type.** `WHERE valueType = 'String'` is rejected, so filter the DataFrame in pandas instead.
- **`commit` is a keyword.** Yield it with backticks: ``CALL db.history() YIELD `commit` ``.
- **No `OPTIONAL CALL`** on procedures.

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
LIST AVAILABLE GRAPHS   // graphs on disk (graphName, isLoaded, isLoading)
CREATE GRAPH g          // new empty graph
LOAD GRAPH g            // load an on-disk graph
CHANGE LIST             // open changes on the current graph
EXPLAIN <query>         // show the compiled plan without running it
```

```python
client.list_available_graphs()  # → list[str]  (json/HTTP backend only)
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
- An unknown hash passed to `checkout(commit=…)` gives `COMMIT_NOT_FOUND`.

After a server restart, past commits must be loaded before they can be read:
- `client.checkout(commit=…)` loads the commit itself.
- `client.set_commit(…)` does **not**: the next query fails with `COMMIT_NOT_LOADED`. Use `checkout(commit=…)`, or load it first.
- Raw REST (`?commit=<hash>`) answers `COMMIT_NOT_LOADED` until you send:
```cypher
LOAD COMMIT 'abc123'
```
A browser client that time-travels should catch `COMMIT_NOT_LOADED`, send `LOAD COMMIT` (it is idempotent) and retry once.

## SDK Client Backends

`TuringDB(...)` is a facade over three interchangeable backends, selected with `type`:

```python
from turingdb import TuringDB

# Default: HTTP/JSON client against a running server
client = TuringDB(host="http://localhost:6666")                  # type="json" implied
client = TuringDB(host="http://localhost:6666", token="s3cret")  # server started with -auth-on

# Native binary protocol (server must run with USE_TURING_PROTO=1, which makes it binary-only)
client = TuringDB(type="native", host="localhost", port=6666)

# In-process engine, no daemon/socket — self-contained, no server to manage
client = TuringDB(type="embedded", data_dir="/path/to/.turing")   # omit data_dir for ~/.turing
```

**Choosing a backend:**
- **`json` (server):** use it by default.
- **`embedded`:** reach for it only when you want a self-contained, in-process engine. It does not support the browser visualizer, `list_available_graphs()`, concurrent multi-process access, S3 transfers, or `token=`.
- **Environment variables:** `type` defaults to the `TURINGDB_TYPE` env var, or `"json"`. `token` defaults to `TURINGDB_AUTH_TOKEN`.
- **Backend-specific methods:** `list_available_graphs()` is json-only. The S3 helpers work on json and native.

## SDK Method Reference

```python
# Construction
TuringDB(type=None, host=..., data_dir=..., port=None, token=None)   # type: "json"|"native"|"embedded"

# Query execution (no parameters argument — inline literals into the string)
client.query(q: str) -> pandas.DataFrame                 # parsed, typed DataFrame
client.query_raw(q: str) -> dict                         # raw server response

# Graph management
client.list_available_graphs() -> list[str]              # json backend only
client.list_loaded_graphs() -> list[str]
client.is_graph_loaded() -> bool
client.create_graph(graph_name: str)
client.load_graph(graph_name: str, raise_if_loaded: bool = True)
client.get_graph() -> str
client.set_graph(graph_name: str)

# Change workflow
client.new_change() -> int                               # CHANGE NEW; ALSO switches the client onto it
client.checkout(change: int | "main" = "main", commit: str = "HEAD")   # call after CHANGE SUBMIT
client.set_change(change: int | str)                     # int or hex string
client.set_commit(commit: str)
# There is no submit() method: submit via client.query("CHANGE SUBMIT"), then client.checkout()

# Connection / timing
client.reconnect()
client.try_reach(timeout: int = 5)
client.warmup(timeout: int = 5)
client.get_query_exec_time() -> float | None             # server-side, ms
client.get_total_exec_time() -> float | None             # round-trip, ms

# S3 / data transfer (json and native backends)
client.s3_connect(bucket_name, access_key=None, secret_key=None, region=None, use_scratch=True)
client.transfer(src, dst)                                # local / s3:// / turingdb:// paths

# Properties
client.current_graph        # active graph
client.current_commit       # "HEAD" by default
client.current_change       # "main" by default
```

### Result types in the DataFrame

- **Nodes and edges** (`RETURN n`, `RETURN r`) come back as integer IDs (`UInt64`). Project properties explicitly.
- **Scalars:** strings → `string`, Int64 / UInt64 → nullable `Int64` / `UInt64`, Double → `float64`, Bool → `boolean`.
- **Temporal:** DateTime → `datetime64[us, UTC]`, Duration → `timedelta64[us]`.
- **Collections:** lists → Python lists, maps → dicts. Embeddings → lists over HTTP, numpy float32 arrays over native/embedded.
- **Paths:** a named path `p` → a list of `{'type': 'node'|'edge', 'id': …}` dicts. A `shortestPath` `path` → a flat list of alternating node/edge IDs.
- **Precision:** over HTTP, doubles are serialized with 6 decimal places.

### Errors

- **Query errors** raise `TuringDBException("<CODE>: <details>")`, where `CODE` is one of `PARSE_ERROR`, `ANALYZE_ERROR`, `PLAN_ERROR`, `EXEC_ERROR`, `CHANGE_NOT_FOUND`, `COMMIT_NOT_FOUND`, …. The details include a caret pointing at the offending part of the query.
- **On the json backend, transport failures** (server down, HTTP 401 for a bad or missing token) surface as `httpx` exceptions instead.
