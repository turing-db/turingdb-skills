---
name: turingdb-introspection
description: Use when exploring an unfamiliar TuringDB graph, inspecting schema, navigating version history, or time-travelling to past commits.
---

# TuringDB: Introspection & Versioning

## Exploring an Unfamiliar Graph

Before writing queries against a graph you don't know, use these `CALL` procedures to understand its shape:

```cypher
CALL db.labels()                  -- all node labels (yields: id, label)
CALL db.edgeTypes()               -- all edge types (yields: id, edgeType)
CALL db.propertyTypes()           -- property keys + value types (yields: id, propertyType, valueType)
CALL db.history()                 -- commit history (yields: commit, nodeCount, edgeCount, partCount)
CALL db.describeCommit('abc123')  -- stats for one commit (yields: nodeCount, edgeCount, partCount)
CALL db.procedures()              -- list available procedures (yields: name, signature)
CALL db.showIndexes()             -- list property indexes (yields: name, size)
```

```python
df = client.query("CALL db.labels()")
df = client.query("CALL db.propertyTypes()")
```

### CALL ... YIELD — chaining procedures into queries

A procedure's output columns can be `YIELD`ed and used by a following `MATCH`/`WHERE`:

```cypher
CALL db.propertyTypes() YIELD *
CALL db.propertyTypes() YIELD propertyType, valueType RETURN *
CALL db.labels() YIELD label, id MATCH (n) WHERE n.name = label RETURN n, label
```

`db.describeCommit` is the only procedure that takes an argument.

### Extensions

Extensions are native libraries that register extra procedure namespaces:

```cypher
INSTALL greeter                              -- load an extension by name
SHOW EXTENSIONS                              -- list installed extensions
SHOW PROCEDURES                              -- list all procedures (same data as CALL db.procedures())
CALL greeter.hello() YIELD message RETURN message
```

## Graph and Change Management

```cypher
LIST GRAPH              -- list all available graphs
CALL db.history()       -- show commit history (there is no bare HISTORY command)
CHANGE LIST             -- list active uncommitted changes
```

```python
client.list_available_graphs()  # → list[str]  (json/HTTP backend only)
client.list_loaded_graphs()     # → list[str]
```

## Versioning / Time Travel

TuringDB stores an immutable commit history. You can query any past state without affecting the current graph.

```python
# Query a specific past commit
client.set_commit("abc123")
df = client.query("MATCH (n) RETURN n")

# Return to current HEAD
client.checkout()   # defaults to main, HEAD

# Switch to a specific change (without checking out)
client.set_change(change_id)

# Switch graph context
client.set_graph("other_graph")
```

Via Cypher (REST API only — CLI and SDK handle this automatically):
```cypher
LOAD COMMIT 'abc123'
```

## SDK Client Backends

`TuringDB(...)` is a facade over three interchangeable backends, selected with `type`:

```python
from turingdb import TuringDB

# Default: HTTP/JSON client against a running server
client = TuringDB(host="http://localhost:6666")      # type="json" implied

# Native binary protocol (faster; needs the compiled extension)
client = TuringDB(type="native", host="localhost", port=6666)

# In-process engine, no daemon/socket — self-contained, no server to manage
client = TuringDB(type="embedded", data_dir="/path/to/.turing")   # omit data_dir for ~/.turing
```

Use the `json` (server) backend by default. The browser visualizer, `list_available_graphs()`, concurrent multi-process access, and S3 transfers all require a server — they are not available in embedded mode. Reach for `embedded` only when you want a self-contained, in-process engine.

`type` also reads the `TURINGDB_TYPE` env var (default `"json"`). `list_available_graphs()` and the S3 helpers are only supported on the `json` backend.

## SDK Method Reference

```python
# Construction
TuringDB(type=None, host=..., data_dir=..., port=None)   # type: "json"|"native"|"embedded"

# Query execution
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
client.new_change() -> int                               # CHANGE NEW; returns change ID
client.checkout(change: int | "main" = "main", commit: str = "HEAD")
client.set_change(change: int | str)                     # int or hex string
client.set_commit(commit: str)
# Note: there is no submit() method — submit via client.query("CHANGE SUBMIT")

# Connection / timing
client.reconnect()
client.try_reach(timeout: int = 5)
client.warmup(timeout: int = 5)
client.get_query_exec_time() -> float | None             # server-side, ms
client.get_total_exec_time() -> float | None             # round-trip, ms

# S3 / data transfer (json backend only)
client.s3_connect(bucket_name, access_key=None, secret_key=None, region=None, use_scratch=True)
client.transfer(src, dst)                                # local / s3:// / turingdb:// paths

# Properties
client.current_graph        # active graph
client.current_commit       # "HEAD" by default
client.current_change       # "main" by default
```

All SDK errors raise `TuringDBException`.
