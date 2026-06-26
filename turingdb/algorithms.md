---
name: turingdb-algorithms
description: Use when running graph algorithms in TuringDB — shortest path (Dijkstra) and vector/embedding similarity search.
---

# TuringDB: Algorithms

## Shortest Path

Uses Dijkstra's algorithm to find the shortest route between nodes, weighted by an edge property.

```cypher
MATCH (a:Station {name: 'Ashchurch'}), (b:Station {name: 'Worcestershire Parkway'})
shortestPath(a, b, distance, dist, path)
RETURN dist, path
```

Signature: `shortestPath(source, target, edge_weight_property, dist_var, path_var)`

- `edge_weight_property` — the edge property to use as the weight. **Bare identifier, not a quoted string** (`distance`, not `'distance'`).
- `dist_var` — output variable for the total distance
- `path_var` — output variable for the path

**Multiple sources/targets:** Pass multiple nodes as source or target — the algorithm returns the best result across all combinations:

```cypher
MATCH (a:Station), (b:Station {name: 'London Euston'})
WHERE a.name = 'Birmingham' OR a.name = 'Manchester'
shortestPath(a, b, distance, dist, path)
RETURN dist, path
```

**Return either or both outputs:**

```cypher
RETURN dist          -- distance only
RETURN path          -- path only
RETURN dist, path    -- both
```

**Gotcha:** Variables used as source or target are consumed by `shortestPath` and **cannot appear in RETURN**. To return properties of the endpoints, match them separately:

```cypher
-- Wrong: a and b cannot appear in RETURN
MATCH (a:Station {name: 'A'}), (b:Station {name: 'B'})
shortestPath(a, b, distance, dist, path)
RETURN a.name, dist   -- ERROR: projection variable a not found in output column

-- Correct: match endpoint data separately (produces a cartesian product with the path output)
MATCH (a:Station {name: 'A'}), (b:Station {name: 'B'}), (start:Station {name: 'A'})
shortestPath(a, b, distance, dist, path)
RETURN start.name, dist, path
```

A join feeding directly into a `shortestPath` endpoint is unsupported (errors with "Common Ancestor Joins With Shortest Path Unsupported").

---

## Vector Search

TuringDB has a built-in k-nearest-neighbor vector index over high-dimensional embeddings. The index lives at the TuringDB root level, independent of graphs and versioning — a single index can serve searches across multiple graphs and commits. Each vector is associated with a numeric ID (typically a node property), keeping the index lightweight.

> **Embedding literals use parentheses `( )`, not brackets.** `(1.2, 2.0, 0.0)` is an embedding; `[1.2, 2.0, 0.0]` is a list literal (a different type). Embedding literals require at least 2 numeric items.

### Setup (one-time, outside a change)

```cypher
CREATE VECTOR INDEX my_index WITH DIMENSION 128 METRIC COSINE
```

- `WITH DIMENSION <n>` — vector dimension (the `WITH` keyword is required)
- `METRIC <metric>` — `COSINE` (cosine similarity / inner product) or `EUCLID` (Euclidean distance). Uppercase keywords.

### Load vectors into the index

Vectors are loaded from a file (relative to the TuringDB `data/` directory):

```cypher
LOAD VECTOR FROM "vectors.csv" IN my_index
```

### Search

`VECTOR SEARCH` finds the k nearest neighbors of a query vector and **yields** their IDs. It is a read statement, so you can chain it with `MATCH`.

```cypher
-- Standalone: find 5 nearest neighbors, return their IDs:
VECTOR SEARCH IN my_index FOR 5 (1.2, 0.5, 3.0, 0.1) YIELD ids
RETURN ids

-- Chain into a graph query — the yielded variable joins like CALL ... YIELD:
VECTOR SEARCH IN my_index FOR 5 (1.2, 0.5, 3.0, 0.1) YIELD ids
MATCH (n:Document) WHERE n.id = ids
RETURN n.title, n.summary
```

Syntax: `VECTOR SEARCH IN <index> FOR <k> (<vector>) YIELD <var>`. The join predicate is `WHERE n.id = ids` — the yielded variable behaves like a column the subsequent `MATCH` binds against.

### Index management

```cypher
SHOW VECTOR INDEXES
DELETE VECTOR INDEX my_index
```

---

## Embeddings as node/edge properties

Separately from the vector index, embeddings can be stored as ordinary node/edge properties (type `Embedding`) and compared with built-in functions. Set them via the normal write workflow (see `writing.md`):

```python
change = client.new_change()
client.checkout(change=change)
client.query("MATCH (n:Person {name: 'Alice'}) SET n.emb = (1.2, 2.0, 0.0, 12.0)")
client.query("CHANGE SUBMIT")
client.checkout()
```

Compare embedding properties with `cosine_similarity` / `euclidean_distance` (both usable in RETURN and WHERE):

```cypher
MATCH (n:Person) RETURN n.name, cosine_similarity(n.emb, (0.4, 0.3, 0.8, 0.1))
MATCH (n:Person) RETURN n.name, euclidean_distance(n.emb, (0.4, 0.3, 0.8, 0.1))
```

### Bulk-loading embedding properties

To attach embeddings to **existing** nodes in bulk, load them from a Parquet file (holding `node_id` int64 + a fixed-width float32 `embedding` column, relative to the `data/` directory). This is a write, so run it inside a change:

```python
change = client.new_change()
client.checkout(change=change)
client.query('LOAD EMBEDDING FROM "embeddings.parquet" AS emb')
client.query("COMMIT")
client.query("CHANGE SUBMIT")
client.checkout()
```

It fails if a `node_id` is missing from the graph, or if the property already exists with a non-embedding type.

When **importing** a graph from JSONL, mark embedding properties at load time instead — array fields named in `WITH EMBEDDINGS` become `Embedding` properties (dimension must be > 1):

```cypher
LOAD JSONL "mydata.jsonl" AS mygraph WITH EMBEDDINGS [{"emb", 384}, {"summary_vec", 768}]
```
