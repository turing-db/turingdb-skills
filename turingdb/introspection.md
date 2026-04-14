---
name: turingdb-introspection
description: Use when exploring an unfamiliar TuringDB graph, inspecting schema, navigating version history, or time-travelling to past commits.
---

# TuringDB: Introspection & Versioning

## Exploring an Unfamiliar Graph

Before writing queries against a graph you don't know, use these procedures to understand its shape:

```cypher
CALL db.labels()           -- all node labels
CALL db.edgeTypes()        -- all edge types
CALL db.propertyTypes()    -- all property keys and their data types
CALL db.history()          -- full commit history
```

```python
df = client.query("CALL db.labels()")
df = client.query("CALL db.propertyTypes()")
```

## Graph and Change Management

```cypher
LIST GRAPH              -- list all available graphs
HISTORY                 -- show commit history
CHANGE LIST             -- list active uncommitted changes
```

```python
client.list_available_graphs()  # → list[str]
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

## Full SDK Method Reference

```python
# Graph management
client.list_available_graphs() → list[str]
client.list_loaded_graphs() → list[str]
client.create_graph(graph_name: str)
client.load_graph(graph_name: str, raise_if_loaded: bool = True)

# Query execution
client.query(query: str) → pandas.DataFrame

# Change workflow
client.new_change() → int                     # returns integer change ID
client.checkout(change: int | str = "main", commit: str = "HEAD")

# Version navigation
client.set_commit(commit: str)
client.set_change(change: int | str)
client.set_graph(graph_name: str)
```

All SDK errors raise `TuringDBException`.
