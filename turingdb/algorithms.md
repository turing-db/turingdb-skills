---
name: turingdb-algorithms
description: Use when running path and graph algorithms in TuringDB v3 — weighted shortest path (Dijkstra statement), hop-count and weighted routes via variable-length paths, vector/embedding similarity search (HNSW/flat indexes), embedding properties and writing computed embeddings back with LOAD EMBEDDING, and GNN neighbourhood sampling procedures.
---

# TuringDB: Algorithms

## Shortest Path (weighted, Dijkstra)

TuringDB has a dedicated `shortestPath` **statement**. It runs Dijkstra between matched nodes, weighted by a numeric edge property:

```cypher
MATCH (a:Station {name: 'Ashchurch'}), (b:Station {name: 'Worcestershire Parkway'})
shortestPath(a, b, distance, dist, path)
RETURN dist, path
```

Signature: `shortestPath(source, target, edge_weight_property, dist_var, path_var)`

- `edge_weight_property` is a **bare identifier, not a quoted string** (`distance`, not `'distance'`). The property must be numeric. Edges where it is null are skipped.
- `dist_var` is the total weight. It is a Double, or an Int64 if the weight is an integer.
- `path_var` is a flat list of alternating IDs: `[node, edge, node, edge, …, node]`.

**Multiple sources/targets:** match several nodes as source and/or target, and the statement returns the single best pair:

```cypher
MATCH (a:Station), (b:Station {name: 'London Euston'})
WHERE a.name = 'Birmingham' OR a.name = 'Manchester'
shortestPath(a, b, distance, dist, path)
RETURN dist, path
```

A join can feed an endpoint, e.g. `MATCH (x {name: 'Bromsgrove'})-[:LINK]->(a), (b {...}) shortestPath(a, b, …)`.

**Restrictions:**
- **It is directed.** Only outgoing edges are followed. An unreachable target returns **zero rows**, not an error.
- **It must be followed directly by `RETURN`.** `WITH` is not allowed after it. `ORDER BY` / `LIMIT` on the RETURN are fine.
- **Only `dist` and `path` can be returned.** Any other variable, including a separately matched copy of an endpoint, fails with `Cannot return a after SHORTESTPATH.` To label the route, map the IDs in `path` back to names with a second query:

```python
row = client.query("MATCH (a:Station {name:'A'}), (b:Station {name:'B'}) "
                   "shortestPath(a, b, distance, dist, path) RETURN dist, path").iloc[0]
node_ids = sorted({int(i) for i in row["path"][0::2]})   # even positions are nodes; distinct
names = client.query(f"UNWIND {node_ids} AS x MATCH (n) WHERE n = x RETURN id(n) AS id, n.name AS name")
# literal list of distinct IDs, x not returned, so this is a constant scan
```

- **Don't call path functions on its output.** `nodes(path)`, `length(path)` and comprehensions over `path` are not supported.
- **The Neo4j/GQL forms are not supported.** `shortestPath((a)-[*]-(b))`, `allShortestPaths(...)`, `ANY SHORTEST` and `SHORTEST k` are all parse errors.

## Path finding with variable-length patterns

For hop-count questions, undirected search, or when you need the nodes and properties along the route, use variable-length paths (full syntax in `querying.md`):

```cypher
// Fewest hops (unweighted), returning the route's names:
MATCH p = (a:Station {name: 'Ashchurch'})-[:LINK*1..6]->(b:Station {name: 'Worcester Shrub Hill'})
RETURN [n IN nodes(p) | n.name] AS route, length(p) AS hops
ORDER BY hops LIMIT 1

// Weighted route, keeping the names along it (enumerates paths: bound the length):
MATCH p = (a:Station {name: 'Ashchurch'})-[:LINK*1..6]->(b:Station {name: 'Worcester Shrub Hill'})
WITH [n IN nodes(p) | n.name] AS route, [r IN relationships(p) | r.distance] AS ds
UNWIND ds AS d
WITH route, sum(d) AS total
RETURN route, total ORDER BY total LIMIT 1

// Reachability / k-hop neighbourhood:
MATCH (a:Person {name: 'Alice'})-[:KNOWS]-{1,3}(b) RETURN DISTINCT b.name
MATCH (a:Person {name: 'Alice'})-[:KNOWS]->+(b) RETURN count(DISTINCT b)
```

- **Paths are enumerated, not searched.** Variable-length matching lists every trail (no repeated edge), which can be expensive on dense graphs. Always bound the length (`*1..6`, `{1,6}`), and prefer the `shortestPath` statement for weighted shortest paths on large graphs.
- **Per-hop filters:** `-[e:LINK WHERE e.distance < 12]->+`, or a quantified pattern `((x)-[e:LINK]->(y) WHERE …){1,4}`.
- **Summing edge properties:** sum relationship properties via `[r IN relationships(p) | r.prop]` + `UNWIND` as above. `UNWIND relationships(p) AS r … sum(r.prop)` fails with an EXEC_ERROR.

---

## Vector Search

TuringDB has a built-in k-nearest-neighbour vector index over high-dimensional embeddings.
- **Scope:** the index lives at the TuringDB root (`<turing-dir>/vector/`), independent of graphs and versioning. One index can serve searches across multiple graphs and commits, and it persists across restarts.
- **Contents:** each vector is stored with a numeric ID, normally the TuringDB node ID.

> **Embedding literals use parentheses `( )`, not brackets.** `(1.2, 2.0, 0.0)` is an embedding; `[1.2, 2.0, 0.0]` is a list literal (a different type). Embedding literals require at least 2 numeric items.

### Setup (one-time, outside a change)

```cypher
CREATE VECTOR INDEX my_index WITH DIMENSION 4 METRIC COSINE                 // real models: 384, 768, 1536…
CREATE VECTOR INDEX my_hnsw WITH DIMENSION 768 METRIC EUCLID TYPE HNSW
```

- `WITH DIMENSION <n>`: the vector dimension. The `WITH` keyword is required, and the maximum is 8192.
- `METRIC COSINE | EUCLID`: uppercase keywords.
  - `COSINE` scores are the **raw inner product**, so **L2-normalize** stored and query vectors to get true cosine similarity.
  - `EUCLID` scores are **squared** L2 distance.
- `TYPE FLAT | HNSW` (optional, default `FLAT`). FLAT is exact; HNSW is an approximate index for large collections.

### Load vectors into the index

Vectors are loaded from a header-less CSV in the `data/` directory. Each line is `id,v1,v2,…,vd`, where `id` is normally the TuringDB node ID (`id(n)`):

```cypher
LOAD VECTOR FROM "vectors.csv" IN my_index
```

To build that file from the graph, export `id(n)` alongside your embeddings, e.g. `MATCH (d:Document) RETURN id(d) AS id, d.text`, embed the text in Python, and write the CSV.

### Search

`VECTOR SEARCH` finds the k nearest neighbours of a query vector and **yields** `ids` (bound as nodes) and `score`. It is a read statement, so you can chain it with `MATCH`, `WHERE` and `RETURN`.

```cypher
// Standalone: top 5 IDs with scores (best first)
VECTOR SEARCH IN my_index FOR 5 (1.2, 0.5, 3.0, 0.1) YIELD ids, score
RETURN ids, score

// ids is a node: read its properties or expand from it directly
VECTOR SEARCH IN my_index FOR 5 (1.2, 0.5, 3.0, 0.1) YIELD ids AS doc, score
WHERE score > 0.8
MATCH (doc)-[:ABOUT]->(t:Topic)
RETURN doc.title, t.name, score

// Slower: re-matching the node scans the label and cross-joins it with the results
VECTOR SEARCH IN my_index FOR 5 (1.2, 0.5, 3.0, 0.1) YIELD ids
MATCH (n:Document) WHERE n = ids
RETURN n.title
```

- **Syntax:** `VECTOR SEARCH IN <index> FOR <k> (<vector>) YIELD ids [AS x][, score [AS s]]`.
- **The query vector's dimension must match the index.** A mismatch is an error.
- **Key the vector file by native node IDs, and use `ids` directly.** `ids` is then the node itself, so `ids.title` and `MATCH (ids)-[...]->(...)` read and expand it without any lookup. Don't re-match it with `MATCH (n:Document) WHERE n = ids` (label scan + cross product), and don't key the file by your own IDs, because joining them back through a property (`WHERE n.doc_id = ids`) is a label scan as well.
- **After `MERGE_DATAPARTS`**, which renumbers nodes, rebuild a node-ID-keyed vector index.
- In the result DataFrame, `ids` is UInt64.

### Index management

```cypher
SHOW VECTOR INDEXES          // name, dimension
DELETE VECTOR INDEX my_index
```

---

## Embeddings as node/edge properties

Separately from the vector index, embeddings can be stored as ordinary node properties (type `Embedding`) and compared with built-in functions.

### Writing embeddings back to nodes: use `LOAD EMBEDDING`

When you compute embeddings for nodes that already exist (e.g. run a model over `d.text` and store the vectors on each `Document`), **write them back with `LOAD EMBEDDING`**. Don't loop `SET n.emb = (...)` queries. Measured on TuringDB 3.0:
- **`LOAD EMBEDDING`:** 20,000 × 384-d embeddings in about **0.1 s**, from one Parquet file.
- **`SET` loop:** about 2.5 ms per node (≈ 50 s for 20k), and every query carries the whole vector as literal text.

The workflow is: read `id(n)`, compute the vectors, write `node_id` + `embedding` to Parquet under `<turing-dir>/data/`, then `LOAD EMBEDDING` inside a change.

```python
import numpy as np, pyarrow as pa, pyarrow.parquet as pq
from pathlib import Path

# 1. Pull the internal node IDs together with the text to embed
docs = client.query("MATCH (d:Document) RETURN id(d) AS node_id, d.text AS text")

# 2. Compute embeddings (any model), as float32
vecs = model.encode(docs["text"].tolist()).astype("<f4")          # shape (n, dim)
vecs /= np.linalg.norm(vecs, axis=1, keepdims=True)               # normalize if you'll use COSINE vector indexes
dim = vecs.shape[1]

# 3. Write node_id + embedding (fixed-size binary of little-endian float32 bytes)
pq.write_table(pa.table({
    "node_id": pa.array(docs["node_id"].astype("int64")),
    "embedding": pa.array([v.tobytes() for v in vecs], pa.binary(dim * 4)),
}), Path.home() / ".turing" / "data" / "doc_embeddings.parquet")   # <turing-dir>/data/

# 4. Load them as property `emb`, inside a change
client.new_change()
client.query('LOAD EMBEDDING FROM "doc_embeddings.parquet" AS emb')   # yields count
client.query("CHANGE SUBMIT")
client.checkout()
```

**File format:**
- `node_id` is the **native TuringDB node ID** (`id(n)`), not a business key. It is stable across submits and restarts, but re-read it after `MERGE_DATAPARTS`, which renumbers nodes.
- `embedding` must be a **fixed-size binary** column (`pa.binary(dim * 4)`). A list or fixed-size-list column is rejected ("missing embedding column").

**Behaviour:**
- **Partial loads:** the file can cover any subset of nodes; only those nodes get the property.
- **Re-running** with the same property name overwrites the stored vectors, which is how you refresh embeddings after a model change.
- **Failures:** the load fails if a `node_id` is not in the graph, or if the property already exists with a non-embedding type.

`SET` is fine for a one-off value:

```python
client.new_change()
client.query("MATCH (n:Person {name: 'Alice'}) SET n.emb = (1.2, 2.0, 0.0, 12.0)")
client.query("CHANGE SUBMIT")
client.checkout()
```

### Comparing embedding properties

Compare embedding properties with `cosine_similarity` (true cosine) and `euclidean_distance` (true, non-squared L2). Both work in RETURN, WHERE and ORDER BY, and both arguments must be Embeddings, not lists:

```cypher
MATCH (n:Person)
RETURN n.name, cosine_similarity(n.emb, (0.4, 0.3, 0.8, 0.1)) AS sim
ORDER BY sim DESC LIMIT 10

MATCH (a:Person {name: 'Alice'}), (b:Person) WHERE a <> b
RETURN b.name, euclidean_distance(a.emb, b.emb) AS d ORDER BY d LIMIT 5
```

This is a brute-force scan. For large collections, use a vector index.

When **importing** a graph from JSONL, mark embedding properties at load time instead. Array fields named in `WITH EMBEDDINGS` become `Embedding` properties (dimension must be > 1):

```cypher
LOAD JSONL "mydata.jsonl" AS mygraph WITH EMBEDDINGS [{"emb", 384}, {"summary_vec", 768}]
```

---

## GNN sampling procedures

Two built-in procedures sample neighbourhoods for graph-neural-network training (signatures via `SHOW PROCEDURES`):

```cypher
// Reservoir-sample up to 2 incoming neighbours of a node (seed optional, for reproducibility)
MATCH (n:Station {name: 'Worcester Shrub Hill'})
CALL gnn.neighbourhoodSample(n, 2, 42) YIELD src, edge, edgeType, tgt
RETURN src.name, edgeType, tgt.name

// GraphSAGE-style 3-hop sampling from seed node IDs with per-hop fanouts
CALL gnn.graphSAGE([8, 12], [10, 5, 5], 7) YIELD *
```

- `gnn.neighbourhoodSample` samples **incoming** edges. A node with no in-edges yields no rows.
- `gnn.graphSAGE(seeds, fanouts, seed)` takes a list of exactly 3 fanouts. It yields `dst_nodes0..2`, `src_nodes0..2` and `tgt_nodes0..2`.

There are no built-in PageRank, community-detection or centrality procedures. Compute those client-side (e.g. export edges with `MATCH (a)-[r]->(b) RETURN id(a), id(b)` into networkx), or approximate them with Cypher aggregation (e.g. degree via `COUNT { (n)--() }`).
