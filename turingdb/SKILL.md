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

TuringDB is a columnar graph database with git-like versioning (changes, commits, time travel). **TuringDB v3** (`turingdb` 3.0 on PyPI) runs every query on a new MLIR-based engine and supports most of openCypher. That covers OPTIONAL MATCH, WITH, UNWIND, UNION, MERGE, DETACH DELETE, REMOVE, variable-length and quantified path patterns, CASE, comprehensions, and EXISTS/COUNT/CALL subqueries. Aggregation uses implicit grouping, and there are DateTime/Duration types. The Python SDK's `query()` returns pandas DataFrames.

## Setup

**Default: connect to a running TuringDB server over HTTP.** The server listens on `http://localhost:6666` by default. If it isn't running, start one with the `turingdb` CLI (see `startup.md`).

```python
from turingdb import TuringDB          # pip/uv package: turingdb>=3.0

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

- **No query parameters.** `$x` gives "Not implemented: Parameters", and `client.query()` takes only a string. Inline literal values into the query text, escaping quotes.
- **Bulk data goes through `LOAD PARQUET`.** Above roughly 5,000 nodes or 1,000 edges, write the data to Parquet and run `LOAD PARQUET`, which builds a new graph in well under a second. Don't loop Cypher CREATE/MERGE or use LOAD CSV + MATCH; MATCH-driven edge creation takes minutes for a few thousand edges. See `importing.md`. To write computed **embeddings back onto existing nodes**, use `LOAD EMBEDDING` from a Parquet file, not `SET` loops; see `algorithms.md`.
- **Writes need a change.** Run them as: `client.new_change()` → writes → `CHANGE SUBMIT` → `client.checkout()`. Inside a change, nodes created by an earlier query are invisible to later queries until you run `COMMIT`. See `writing.md`.
- **Address known nodes by native node ID, written as a literal: `MATCH (n) WHERE n = 123`.** That is a constant scan of exactly that node. For a batch, inline the IDs from Python as a literal list of distinct integers: `UNWIND [4, 17, 42] AS x MATCH (n) WHERE n = x`, and don't return `x`. That costs about 1.4 µs per node on 1M nodes.
  - **Slow forms:** looking nodes up through a user-level key property (`{pid: x}` per row) is about 800× slower. `id(n) = 123`, `n IN [...]`, and IDs from a variable or `collect()` scan every node.
  - **Never compute IDs per row (`a = ss[i]`), and never look up two nodes from one row (`a = p[0] AND b = p[1]`).** That runs as a cross product of full scans and can exhaust server memory. See `querying.md`.
  - **Keep each `UNWIND` batch to at most 5,000 IDs.** Around 5,500 literals the planner's rewrite overflows its stack and the server crashes (it logs nothing). Split larger lists across queries.
- **Boolean chains are capped at 255 terms.** A `WHERE` with 256 `OR`/`AND` terms fails with `PARSE_ERROR: Expression nested deeper than 256 levels`. Use `x IN [...]` for long lists.
- **`RETURN n` returns an integer node ID, not a property map.** There is no `properties(n)` or `n{.*}`, and `keys()` doesn't work. Return the properties you need explicitly (`n.name, n.age`).
- **No string or math function library.** `toUpper`, `substring`, `split`, `replace`, `trim`, `abs`, `round`, `sqrt` and `rand` do not exist. Neither do `=~` regex or `reduce()`. Filter strings with `STARTS WITH` / `ENDS WITH` / `CONTAINS`, and do the rest in pandas.
- **Maps can be built and stored, but not read.** `m.key`, `m['key']` and map projection `n{.name}` are not supported.
- **Labels are fixed at creation.** `SET n:Label` and `REMOVE n:Label` are unsupported. Every node needs at least one label, and every edge exactly one type.
- **Shortest path:** only TuringDB's statement `shortestPath(a, b, weightProp, dist, path)` exists. `shortestPath((a)-[*]-(b))` does not. See `algorithms.md`.
- **Strict numeric equality.** `Integer = Double` and `Double = Double` are rejected. Use `<` / `>` ranges or `toInteger`.
- **Function names are case-sensitive** (`avg`, not `AVG`), except `count` and `collect`. Keywords are case-insensitive.
- **One statement per request.** `;`-separated scripts are rejected. Comments are `//` only; `--` is not a comment.

## Routing

Based on what the user is asking, immediately read the matching file from this same directory using the Read tool. Do not ask the user which file to read — determine it from context.

| Task | File |
|------|------|
| Install, start/stop a server, connect (HTTP, auth, embedded), load/create a graph | `startup.md` |
| Reading data: MATCH, OPTIONAL MATCH, WHERE, paths, WITH, aggregation, UNWIND, subqueries, functions, LOAD CSV | `querying.md` |
| Writing data: CREATE, MERGE, SET, REMOVE, DELETE, the change/commit workflow, indexes | `writing.md` |
| Importing external data: CSV, JSONL, GML, Parquet (`LOAD PARQUET` and `turing-parquet`), Neo4j migration | `importing.md` |
| Paths and algorithms: shortest path, variable-length paths, vector/embedding search, writing embeddings back (`LOAD EMBEDDING`), distributed GNN training with `gnn.graphSAGE` sampling | `algorithms.md` |
| Exploring an unfamiliar graph, procedures, versioning/time travel, SDK reference | `introspection.md` |

If the task spans multiple areas (e.g. connect then query), read the files in sequence.
