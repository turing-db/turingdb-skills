---
name: turingdb-algorithms
description: Use when running graph algorithms in TuringDB — shortest path (Dijkstra) and vector/embedding similarity search.
---

# TuringDB: Algorithms

## Shortest Path

Uses Dijkstra's algorithm to find the shortest route between nodes, weighted by an edge property.

```cypher
MATCH (a:Station {name: 'Ashchurch'}), (b:Station {name: 'Worcestershire Parkway'})
shortestPath(a, b, 'distance', dist, path)
RETURN dist, path
```

Signature: `shortestPath(source, target, edge_weight_property, dist_var, path_var)`

- `edge_weight_property` — the edge property to use as the weight (string)
- `dist_var` — output variable for the total distance
- `path_var` — output variable for the path

**Multiple sources/targets:** Pass multiple nodes as source or target — the algorithm returns the best result across all combinations:

```cypher
MATCH (a:Station), (b:Station {name: 'London Euston'})
WHERE a.name IN ['Birmingham', 'Manchester']
shortestPath(a, b, 'distance', dist, path)
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
shortestPath(a, b, 'distance', dist, path)
RETURN a.name, dist   -- ERROR

-- Correct: match endpoint data separately
MATCH (a:Station {name: 'A'}), (b:Station {name: 'B'})
shortestPath(a, b, 'distance', dist, path)
MATCH (start:Station {name: 'A'})
RETURN start.name, dist, path
```

---

## Vector Search

TuringDB has built-in k-nearest-neighbor search over high-dimensional embeddings. Vectors are stored as node/edge properties and indexed separately.

### Setup (one-time, outside a change)

```cypher
CREATE VECTOR INDEX my_index DIMENSION 128 METRIC cosine
```

Metric options: `cosine`, `euclidean`

### Store embeddings on nodes

Embeddings are set via the normal write workflow:

```python
change = client.new_change()
client.checkout(change=change)
client.query("MATCH (n:Person {name: 'Alice'}) SET n.emb = [1.2, 2.0, 0.0, ...]")
client.query("CHANGE SUBMIT")
client.checkout()
```

### Search

```cypher
-- Find 5 nearest neighbors, returns their IDs:
VECTOR SEARCH my_index [1.2, 0.5, 3.0, ...] LIMIT 5

-- Chain results into a graph query:
VECTOR SEARCH my_index [1.2, 0.5, 3.0, ...] LIMIT 5 AS ids
MATCH (n) WHERE n.id IN ids RETURN n

-- Return neighbor properties alongside graph data:
VECTOR SEARCH my_index [1.2, 0.5, 3.0, ...] LIMIT 5 AS ids
MATCH (n)-[:AUTHORED]->(p:Publication)
WHERE n.id IN ids
RETURN n.name, p.title
```

### Index management

```cypher
SHOW VECTOR INDEXES
DELETE VECTOR INDEX my_index
```
