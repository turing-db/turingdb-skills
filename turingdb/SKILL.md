---
name: turingdb
description: >
  Start, query, write, and manage TuringDB columnar graph databases using the Python SDK and Cypher dialect.
  TRIGGER when: code imports `turingdb` or `TuringDB`; user mentions TuringDB, turing db, or turing database;
  user asks about TuringDB Cypher queries, graph versioning with changes/commits, vector search in TuringDB,
  or the `turingdb` CLI; files contain TuringDB connection strings (localhost:6666) or TuringDB SDK calls
  (`client.query`, `client.new_change`, `client.load_graph`).
  SKIP: generic graph/Neo4j/Cypher questions with no TuringDB mention; general database work unrelated to TuringDB.
---

# TuringDB

TuringDB is a columnar graph database with git-like versioning. Its query language is a Cypher dialect — largely compatible with standard Cypher but with TuringDB-specific extensions. The Python SDK `query()` method returns results as pandas DataFrames.

## Setup

**Default: connect to a running TuringDB server over HTTP.** The server listens on `http://localhost:6666` by default; start one with the `turingdb` CLI if it isn't already running (see `startup.md`).

```python
from turingdb import TuringDB

client = TuringDB(host="http://localhost:6666")   # HTTP/JSON client (the default backend)
client.create_graph("my_graph")  # create new graph
# or
client.load_graph("my_graph")    # load an existing on-disk graph by name
client.set_graph("my_graph")     # make it active
```

`TuringDB(...)` supports three backends via `type`: `"json"` (HTTP, the default — pass `host="http://localhost:6666"`), `"native"` (binary protocol over HTTP — `host=`/`port=`), and `"embedded"` (in-process, no server — pass `data_dir=`). Use the **server** (`json`) by default; it's required for the browser visualizer (`-ui`), `list_available_graphs()` to discover on-disk graphs, concurrent multi-process access, and S3 transfers. Reach for `"embedded"` only when you want a self-contained, in-process engine with no server to manage. See `startup.md` for connecting/starting the server and `introspection.md` for the full backend reference.

## Routing

Based on what the user is asking, immediately read the matching file from this same directory using the Read tool. Do not ask the user which file to read — determine it from context.

| Task | File |
|------|------|
| Connect to a TuringDB server (or run embedded) and load/create a graph | `startup.md` |
| Reading data — MATCH, WHERE, filtering, traversal, joins | `querying.md` |
| Writing data — CREATE, SET, updating the graph | `writing.md` |
| Importing external data — JSONL, GML, Parquet, Neo4j migration | `importing.md` |
| Graph algorithms — shortest path, vector/embedding search | `algorithms.md` |
| Exploring an unfamiliar graph, versioning, time travel, GML/JSONL import | `introspection.md` |
| Importing Parquet files via the `turing-parquet` CLI (turingdb ≥ 1.32) | `parquet.md` |

If the task spans multiple areas (e.g. connect then query), read the files in sequence.
