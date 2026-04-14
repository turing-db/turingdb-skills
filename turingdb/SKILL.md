---
name: turingdb
description: Entrypoint for all TuringDB work. Use this first to get oriented, then follow the pointer to the right skill for your task.
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
