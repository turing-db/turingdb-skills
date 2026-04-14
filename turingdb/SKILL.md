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

```python
from turingdb import TuringDB

client = TuringDB(host="http://localhost:6666")   # full URL, not separate host/port
client.load_graph("my_graph")    # load existing graph
# or
client.create_graph("my_graph")  # create new graph
```

Start the server with `turingdb start` (foreground) or `turingdb start -demon` (background). Default port is 6666. Use `-turing-dir <path>` to point at a specific data directory.

## Routing

Based on what the user is asking, immediately read the matching file from this same directory using the Read tool. Do not ask the user which file to read — determine it from context.

| Task | File |
|------|------|
| Start the server, connect, and load a graph | `startup.md` |
| Reading data — MATCH, WHERE, filtering, traversal, joins | `querying.md` |
| Writing data — CREATE, SET, updating the graph | `writing.md` |
| Graph algorithms — shortest path, vector/embedding search | `algorithms.md` |
| Exploring an unfamiliar graph, versioning, time travel | `introspection.md` |

If the task spans multiple areas (e.g. start the server then query it), read the files in sequence.
